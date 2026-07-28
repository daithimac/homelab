---
type: Playbook
title: 133gsl.ie on Cloudflare DNS, with Let's Encrypt wildcard certs
description: How the .ie domain was delegated to Cloudflare, why registration stayed at Maxer, and how Caddy now issues a publicly-trusted *.133gsl.ie wildcard over DNS-01 without opening a single port.
resource: 133gsl.ie
tags: [dns, cloudflare, tls, letsencrypt, acme, caddy, adguard, split-horizon, ie]
timestamp: 2026-07-28T00:00:00Z
---

Dave bought `133gsl.ie` from Maxer and wanted it fronting the homelab. Set up 2026-07-28.
The outcome: every service now answers on `https://<name>.133gsl.ie` with a **real
Let's Encrypt certificate**, while remaining reachable only from inside the tailnet —
nothing is published to the internet and no port is forwarded.

This retires the per-device chore of trusting Caddy's internal CA
([reverse-proxy-caddy.md](reverse-proxy-caddy.md#trusting-the-internal-ca)) for anything on
the `.ie` name.

# The three facts that shaped the design

* **Cloudflare Registrar does not sell `.ie`.** Attempting a transfer returns *".ie domains
  aren't supported yet"* with an **Add site anyway** button. Registration stays at Maxer;
  only DNS moves. This is not a limitation in practice — Cloudflare's free DNS works fully
  with an external registrar.
* **`.ie` runs a pre-delegation zonecheck.** The registry refuses a nameserver change unless
  the new nameservers already answer authoritatively for the domain. The Cloudflare zone
  must therefore exist **before** the change is made at Maxer — the reverse of the usual
  `.com` order.
* **The `.ie` zone reloads only on odd hours** (01, 03, 05 … 23). Delegation is never
  instant. In practice it landed in ~40 minutes.

# Split-horizon: the public zone is deliberately almost empty

| Where | What it holds |
|---|---|
| **Cloudflare (public)** | one `CAA` record. No A, no AAAA, no MX, no `www`. |
| **AdGuard (internal)** | one wildcard rewrite, `*.133gsl.ie` → `192.168.0.14` |

Publishing `jellyfin → 192.168.0.12` in the public zone is the tempting mistake: it leaks
the internal topology to anyone who queries, and Cloudflare cannot proxy RFC1918 addresses
anyway. **The emptiness of the public zone is the security boundary.** Verify it stays that
way:

```bash
dig +short A jellyfin.133gsl.ie @1.1.1.1        # must be empty
dig +short A jellyfin.133gsl.ie @192.168.0.20   # must be 192.168.0.14
```

A **wildcard certificate is used rather than per-service certs on purpose.** Every
publicly-trusted certificate is published to Certificate Transparency logs, so 15
individual certs would publish `sabnzbd.133gsl.ie`, `proxmox.133gsl.ie` and the rest as a
public, greppable inventory of the lab. `*.133gsl.ie` leaks only the domain.

# Setup as performed

## 1. Cloudflare zone, before touching the registrar

Added the domain via **Connect a domain** (not Transfer), Free plan, Full setup. The scan
found **0 records** — the domain was freshly registered and never delegated, so there was
no DNS to migrate at all. Added one `CAA` record: `@`, flags `0`, tag `issue`, CA domain
`letsencrypt.org`.

Confirmed authoritative before proceeding, because this is exactly what the registry checks:

```bash
dig @cris.ns.cloudflare.com 133gsl.ie SOA +norecurse +noall +comments +answer
```

Wanted `flags: qr aa` with `ANSWER: 1`.

## 2. Universal SSL turned off — otherwise the CAA record is a lie

Cloudflare **silently appends CAA records for all five of its own Universal SSL CA
providers** (comodoca, digicert, letsencrypt, pki.goog, ssl.com, plus matching `issuewild`)
the moment any CAA record exists on a zone with Universal SSL enabled. They are served in
DNS but **hidden from the dashboard's DNS list**, so the UI shows 1 record and `dig` shows
10.

Since nothing here is proxied through Cloudflare, Universal SSL was doing nothing, so it was
disabled (**SSL/TLS → Edge Certificates → Universal SSL**) and the appended records vanished.
The zone is back to a single `0 issue "letsencrypt.org"`.

No `issuewild` record is needed — under RFC 8659, when `issuewild` is absent the `issue`
rule governs wildcards too.

## 3. Nameservers at Maxer

Maxer → Manage → Nameservers → **Use custom nameservers**: `cris.ns.cloudflare.com` and
`dawn.ns.cloudflare.com`, slots 3–5 left blank. DNSSEC confirmed off first (a stale DS
record while Cloudflare serves the zone breaks resolution outright). Registrar Lock left on.

## 4. Caddy rebuilt with the Cloudflare DNS module

The stock `caddy:2` image has **no DNS providers compiled in**, so DNS-01 is simply not
available until you rebuild. On [docker-stack (VM103)](/containers/docker-stack.md):

```dockerfile
FROM caddy:2-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/cloudflare
FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

Built as `caddy-cf:2` from `/data/caddy/build`. **Confirm the module actually landed** —
if this is empty, everything downstream fails with errors that never point back here:

```bash
ssh dave@192.168.0.14 'docker run --rm caddy-cf:2 caddy list-modules | grep dns.providers.cloudflare'
```

`docker-compose.yml`'s `caddy` service changed to `image: caddy-cf:2` plus
`env_file: /opt/stack/caddy.env`. The new image was brought up **before** any `.ie` config
was written, so that "is the rebuilt binary a clean drop-in?" was answered on its own.

## 5. The API token

Cloudflare token scoped to `133gsl.ie` only, with exactly two permissions:

| Permission | Level | Why |
|---|---|---|
| **DNS** | Edit | write the `_acme-challenge` TXT record |
| **Zone** | Read | resolve the zone ID by name before touching records |

`Zone → Edit` is the wrong choice — it grants zone *management* and still doesn't provide
what ACME needs. The permission called **DNS** is the one that matters, and searching the
picker for "zone" hides it.

Stored at `/opt/stack/caddy.env`, mode `600`, owned by `dave`. Two things forced that
location:

* `env_file` is read by the **docker CLI as the invoking user**, not by the daemon — a
  `600 root:root` file makes `docker compose` fail for a non-root user.
* `/data/caddy` is bind-mounted to `/etc/caddy`, so a token left there sits inside Caddy's
  own config directory. `/opt/stack/` keeps it out.

## 6. The Caddyfile block

Appended to `/data/caddy/Caddyfile` (backup: `Caddyfile.bak-20260728-preie`):

```
*.133gsl.ie {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1 1.0.0.1
    }

    @jellyfin host jellyfin.133gsl.ie
    handle @jellyfin {
        reverse_proxy 192.168.0.12:8096
    }
    # … one @matcher + handle pair per service …

    @proxmox host proxmox.133gsl.ie
    handle @proxmox {
        reverse_proxy 192.168.0.10:8006 {
            transport http {
                tls
                tls_insecure_skip_verify
            }
        }
    }

    handle {
        respond "no such service on 133gsl.ie" 404
    }
}
```

Then `caddy validate` (wanted `Valid configuration`) and `caddy reload`.

**`resolvers 1.1.1.1 1.0.0.1` is mandatory, not decoration.** AdGuard's wildcard rewrite
answers for *every* name under the domain — including `_acme-challenge.133gsl.ie`. Without
an explicit external resolver, Caddy's DNS-01 propagation self-check asks AdGuard, believes
the rewrite, and issuance never completes.

# What's on the .ie name

Same backends as the `.lan` table in
[reverse-proxy-caddy.md](reverse-proxy-caddy.md#whats-proxied): jellyfin, openwebui, ollama,
sabnzbd, adguard, grafana, sillytavern, audiobooks, audiobookshelf, coder, n8n, qdrant,
kokoro, proxmox.

Two deliberate differences from the `.lan` set:

* **`opencode` was not carried over.** Nothing listens on `192.168.0.18:8080`; the `.lan`
  entry is a known-dead one already filed in [actions.md](/actions.md), and duplicating it
  would just double the stale entry.
* **The `.docker.lan` names flatten to one label.** `n8n.docker.lan` becomes
  `n8n.133gsl.ie`, not `n8n.docker.133gsl.ie` — **a wildcard certificate covers exactly one
  label**, so `*.133gsl.ie` would not match a two-label name and issuance would fail for it.

`.lan` names are unchanged and still work. Keep both in parallel: if the Cloudflare token or
the zone ever breaks, `.lan` with the internal CA is the fallback and costs nothing.

# Gotchas hit during setup (2026-07-28)

* **`*` must be quoted in AdGuardHome.yaml.** The rewrite has to be written
  `- domain: '*.133gsl.ie'`. An unquoted `*` starting a YAML scalar is an **alias
  reference**, and AdGuardHome would have crash-looped on restart exactly as it once did
  over the 4-space/6-space indentation trap documented in
  [DNS via AdGuard](/network/dns-adguard.md).
* **`ns1.maxer.com` is not authoritative for Maxer-registered domains.** An early
  `dig +short ... @ns1.maxer.com` returned empty for every record type, which reads
  identically to "the zone is empty". It was actually the server declining to answer — the
  real backend is `ns1.fastsecurehost.com` (`217.115.112.133`). **Always check for the `aa`
  flag rather than trusting `+short`'s silence.**
* **Cloudflare has two kinds of API token with different verify endpoints.** An
  account-owned token (Manage Account → API Tokens) returns `Invalid API Token` from
  `/user/tokens/verify` while being perfectly valid — it verifies at
  `/accounts/{account_id}/tokens/verify`. Roughly 40 minutes were lost to treating a working
  token as broken. **Test the token against the calls the client actually makes** rather
  than a generic verify endpoint:

  ```bash
  # zone lookup, then create and delete a TXT — exactly what libdns/cloudflare does
  curl -s -H "Authorization: Bearer $CF_API_TOKEN" \
    "https://api.cloudflare.com/client/v4/zones?name=133gsl.ie"
  ```

  Account-owned tokens work fine with `caddy-dns/cloudflare`; only the verify URL differs.
* **`qm guest exec … | grep` does not filter.** The guest's stdout comes back as a single
  JSON-escaped string, so a host-side pipe matches the whole blob. Put the pipe *inside* the
  guest command. For editing files on VM103, SSH into the VM directly instead — `qm guest
  exec` is a poor channel for anything multi-line.
* **SABnzbd's `host_whitelist` bites again on every new hostname.** `sabnzbd.133gsl.ie` had
  to be added to `/root/.sabnzbd/sabnzbd.ini` on [CT108](/containers/sabnzbd.md), same as
  `sabnzbd.lan` before it. The unit is **`sabnzbdplus@root.service`**, not `sabnzbd` —
  restarting the wrong name silently succeeds and changes nothing.

# Verification

```bash
# publicly-trusted wildcard, not Caddy's internal CA
echo | openssl s_client -connect 192.168.0.14:443 -servername jellyfin.133gsl.ie 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates -ext subjectAltName

# note: no -k. If this passes, the chain validates against the system trust store.
curl -o /dev/null -w "%{http_code}\n" --resolve jellyfin.133gsl.ie:443:192.168.0.14 \
  https://jellyfin.133gsl.ie/
```

Confirmed 2026-07-28: issuer `C=US, O=Let's Encrypt, CN=YE1`, subject `CN=*.133gsl.ie`,
SAN `DNS:*.133gsl.ie`, valid to 2026-10-26. Nine services returned `200`/`302` **without
`-k`**. All `.lan` names re-tested and unaffected. Public DNS returns nothing for any
service name.

Renewal is automatic and needs no port — Caddy re-runs DNS-01 against the same token. **The
token is the single point of failure for renewal**: it has no expiry set, deliberately, so
that certificates don't silently stop renewing months from now.

# Citations

[1] Live setup and verification, this homelab (2026-07-28) — Cloudflare dashboard, Maxer
    control panel, and direct `ssh`/`pct exec`/`docker exec` work on the host and VM103.
[2] Cloudflare docs — [Add CAA records](https://developers.cloudflare.com/ssl/edge-certificates/caa-records/),
    [CAA FAQ](https://developers.cloudflare.com/ssl/edge-certificates/troubleshooting/caa-records/).
[3] IE Domain Registry — [Registration and Naming in the .IE Namespace](https://www.iedr.ie/uploads/IEDR-RegistrationNaming-.IE-Namespace.pdf).
