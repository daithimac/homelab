---
type: Playbook
title: Reverse proxy via Caddy (on docker-stack)
description: How the Caddy reverse proxy on VM103 gives every .lan service real HTTPS with no port in the URL, how to add a new domain, and the two gotchas that bit during setup.
tags: [caddy, reverse-proxy, tls, dns, docker-stack, adguard]
timestamp: 2026-07-25T00:00:00Z
---

A `caddy:2` container (`stack-caddy-1`) already existed on
[docker-stack (VM103)](/containers/docker-stack.md), fully configured with automatic
internal HTTPS (`tls internal`) for nearly every `.lan` domain — but it was never wired
into DNS, so it sat unused. Discovered 2026-07-25 while fixing `https://jellyfin.lan` not
resolving (AdGuard's rewrites pointed straight at each service's own IP, bypassing Caddy
entirely). Wired up the same day rather than building a second, redundant proxy.

# How it works

* Config: `/data/caddy/Caddyfile` on the **docker-stack VM's own filesystem** (bind-mounted
  into the container at `/etc/caddy`, not baked into the image) — edit the host path
  directly.
* Data (including the internal CA): `/data/caddy/data`, bind-mounted to `/data` in the
  container.
* Caddy listens on the VM's `80`/`443` (`192.168.0.14`). Each `.lan` domain gets its own
  block with `tls internal` (Caddy's built-in local CA — no manual cert management) and a
  `reverse_proxy` directive pointing at the real backend `ip:port`.
* AdGuard's rewrite for each proxied domain points at **`192.168.0.14`** (docker-stack),
  not the backend's own IP. Caddy picks the right backend via SNI/Host header.

# What's proxied

| Domain | Backend |
|---|---|
| `jellyfin.lan` | `192.168.0.12:8096` |
| `openwebui.lan` | `192.168.0.17:8080` |
| `ollama.lan` | `192.168.0.13:11434` |
| `sabnzbd.lan` | `192.168.0.19:8080` |
| `adguard.lan` | `192.168.0.20:80` |
| `grafana.lan` | `192.168.0.21:3000` |
| `sillytavern.lan` | `192.168.0.22:8000` — added 2026-07-25, wasn't in the original config |
| `proxmox.lan` | `192.168.0.10:8006` (has its own websocket transport block for the console) |
| `n8n.docker.lan` | `n8n:5678` (Docker service name, same Compose network) |
| `qdrant.docker.lan` | `qdrant:6333` — block existed in Caddy but had **no** AdGuard rewrite at all until 2026-07-25; added |
| `kokoro.docker.lan` | `kokoro:8880` — added 2026-07-26 with the [Kokoro TTS service](/containers/docker-stack.md#kokoro-tts-added-2026-07-26) |
| `audiobooks.lan` | `192.168.0.24:7860` — added 2026-07-26, the [epub_to_audiobook Gradio UI](/containers/audiobooks.md#web-ui--audiobook-uiservice) |
| `audiobookshelf.lan` | `192.168.0.26:13378` — added 2026-07-26, the [Audiobookshelf server (CT112)](/containers/audiobookshelf.md). **Different service from `audiobooks.lan`** — that one converts, this one plays |
| `coder.lan` | `192.168.0.27:8080` — added 2026-07-26, [code-server (CT113)](/containers/code-server.md). The one domain that isn't its guest's hostname (`code-server`) |
| `opencode.lan` | `192.168.0.18:8080` — **dead**, nothing listens on that port; see below |

**Not proxied** (no HTTP service to front): `nas.lan` (Samba only), `tailscale.lan` (no web
UI).

`opencode.lan` used to be listed here as "not proxied — CLI tool, nothing listens". That
was half right. There is **no service listening** on `192.168.0.18:8080` (re-confirmed
2026-07-26, `curl` from the host times out), but a Caddy block and an AdGuard rewrite for
it both exist regardless — so the domain resolves and then fails at the proxy rather than
at DNS. Harmless, but it's a stale entry rather than an absent one. Filed in
[actions.md](/actions.md).

Gradio apps like `audiobooks.lan` need no special websocket config — a plain
`reverse_proxy` handles them, unlike the `proxmox.lan` console which does need its own
`transport http` block. The same held for Audiobookshelf, which is websocket-heavy
(socket.io) and still needed nothing beyond a plain `reverse_proxy` — verified end-to-end
2026-07-26, HTTP 200 through the proxy. **`coder.lan` is the case where this was proven
rather than assumed**: a raw websocket handshake through Caddy returned `101 Switching
Protocols` and the backend's first frame decoded as a real VS Code `Resume` control message,
with an identical result direct to the backend — so the proxy demonstrably isn't interfering
with the upgrade. If you ever need to check a websocket app this way, that probe is
described in [code-server (CT113)](/containers/code-server.md#verification--including-the-websocket).

Note the naming split: services running **on docker-stack itself** get
`<name>.docker.lan` and are proxied by Docker service name (`kokoro:8880`); services on
their **own guest** get `<name>.lan` and are proxied by IP:port.

# Trusting the internal CA

Caddy's root CA lives at
`/data/caddy/data/caddy/pki/authorities/local/root.crt` on docker-stack. Fetch it
yourself and trust it in your OS/browser keychain — this is a one-time, per-device,
security-relevant action nobody else should do on your behalf:

```bash
qm guest exec 103 -- cat /data/caddy/data/caddy/pki/authorities/local/root.crt
```

Until a device trusts it, `https://*.lan` there will show a certificate warning (the proxy
and TLS still work — it's just self-signed from the browser's point of view).

# Adding a new domain

1. Add a block to `/data/caddy/Caddyfile` on docker-stack:
   ```
   newservice.lan {
       tls internal
       reverse_proxy 192.168.0.XX:PORT
   }
   ```
2. **Validate before reloading** — catches a bad block without touching the running proxy:
   `qm guest exec 103 -- docker exec stack-caddy-1 caddy validate --config /etc/caddy/Caddyfile`
   (want `Valid configuration`; it also logs some `servers shutting down` noise from the
   throwaway validation instance, which is normal and not your proxy restarting).
   Then reload without downtime:
   `qm guest exec 103 -- docker exec stack-caddy-1 caddy reload --config /etc/caddy/Caddyfile`
   A `Caddyfile input is not formatted` warning pointing at line 2 is pre-existing and
   unrelated to whatever you just added.
3. Point AdGuard's rewrite at `192.168.0.14` (see [DNS via AdGuard](/network/dns-adguard.md)
   for the rewrite-editing procedure) and `systemctl restart AdGuardHome` — a `pct set
   -nameserver`-style live-apply does **not** exist for rewrites; restart is required.
4. Test end-to-end before trusting it: `dig +short <domain> @192.168.0.20` should return
   `192.168.0.14`, then `curl -sk --resolve <domain>:443:<that-ip> https://<domain>/`.

# Gotchas hit during setup (2026-07-25)

* **AdGuardHome.yaml's `rewrites:` list items are indented 4 spaces for `- domain:`, 6 for
  `answer:`/`enabled:`** — not the 2/4 it looks like at a glance. Get this wrong and
  AdGuardHome crash-loops on restart with `go-yaml load error ... did not find expected
  key`, and the reported line range is unhelpfully far from the actual mistake. Measure
  precisely (`awk 'match($0,/^ */){print RLENGTH}'`) rather than eyeballing an existing
  entry — don't trust a quick visual read of indentation in a screenshot or terminal.
* **Self-hosted apps behind a reverse proxy can 403 on Host-header whitelisting.** SABnzbd
  ships a `host_whitelist` setting (`/root/.sabnzbd/sabnzbd.ini` on
  [sabnzbd (CT108)](/containers/sabnzbd.md)) that only allowed the bare `sabnzbd` hostname —
  requests arriving via Caddy with `Host: sabnzbd.lan` got a `403` until `sabnzbd.lan` was
  added to that list and the service restarted. Check for an equivalent setting first if a
  newly-proxied app 403s instead of loading.

# Citations

[1] Live `qm guest exec 103` / `pct exec` investigation and fix, this homelab (2026-07-25).
