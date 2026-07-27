---
type: VM
title: docker-stack (VM103)
description: Ubuntu VM running Postgres, Qdrant, n8n, Kokoro TTS, and a Caddy reverse proxy in Docker, at 192.168.0.14.
tags: [proxmox, vm, docker, postgres, qdrant, n8n, caddy, reverse-proxy, kokoro, tts]
timestamp: 2026-07-26T00:00:00Z
---

VM103, an Ubuntu VM running a Docker stack: **Postgres**, **Qdrant**, **n8n**, **Kokoro**
(TTS, added 2026-07-26) and (found running but completely undocumented until 2026-07-25)
**Caddy** — a reverse proxy fronting most other guests' web UIs with automatic HTTPS. See
[Reverse proxy via Caddy](/playbooks/reverse-proxy-caddy.md).

# IP — confirmed static

**192.168.0.14**, confirmed static via the guest's own netplan config on 2026-07-25
(queried through the QEMU guest agent, `qm guest exec 103 -- grep -r . /etc/netplan/`, no
SSH needed):

```yaml
network:
  version: 2
  ethernets:
    enp6s18:
      addresses:
      - "192.168.0.14/24"
      nameservers:
        addresses:
        - 192.168.0.20   # was 192.168.0.1 (the router) — fixed same day, see below
        search: []
      routes:
      - to: "default"
        via: "192.168.0.1"
```

The address is hardcoded (no `dhcp4: true`), so unlike home-assistant's old `.213` this
one isn't at risk of drifting on lease renewal — `.14` sitting in the historical
DHCP-collision zone (`.12/.14/.16` churn, see [IP addressing](/network/ip-addressing.md))
is a coincidence of history, not an active conflict.

**DNS fixed 2026-07-25.** Nameserver was the router (`192.168.0.1`), not AdGuard
(`192.168.0.20`) — same bypass pattern home-assistant had. Fixed via the QEMU guest agent
(no SSH needed): patched `/etc/netplan/50-cloud-init.yaml`'s `nameservers.addresses` to
`192.168.0.20` with a precise `sed` (targeted the unquoted `- 192.168.0.1` nameserver
line, left the quoted `via: "192.168.0.1"` gateway/route line untouched), then
`netplan apply`. Verified with `resolvectl status enp6s18`: DNS Servers now
`192.168.0.20`.

**Unrelated discovery made while verifying this fix**: querying `jellyfin.lan` from this
VM returned `192.168.0.14` (this VM's own IP) instead of jellyfin's real `.12`. Root
cause is **not** this VM — it's a pre-existing bug in AdGuard's rewrite table, where 7 of
12 domain rewrites were all miscopied to point at `.14`. See
[DNS via AdGuard](/network/dns-adguard.md) and [actions.md](/actions.md).

# Reverse proxy (Caddy)

`stack-caddy-1` (image `caddy:2`) listens on this VM's `80`/`443` and reverse-proxies most
other guests' web UIs with automatic internal HTTPS — `https://jellyfin.lan`,
`https://grafana.lan`, etc. all route through here rather than hitting each service's own
IP directly. Full domain table, how to add a new one, and the two gotchas hit while wiring
it up (an AdGuardHome YAML indentation trap and a SABnzbd Host-header 403): see
[Reverse proxy via Caddy](/playbooks/reverse-proxy-caddy.md).

# Kokoro TTS (added 2026-07-26)

`stack-kokoro-1` (image `ghcr.io/remsky/kokoro-fastapi-cpu:latest`, 5.42GB) serves
**Kokoro-82M** text-to-speech on `192.168.0.14:8880` with an OpenAI-compatible API
(`/v1/audio/speech`), plus a voice-auditioning web UI at `/web` and API docs at `/docs`.
Reverse-proxied as `https://kokoro.docker.lan`.

It exists to feed [audiobooks (CT111)](/containers/audiobooks.md) — see
[EPUB/PDF to audiobook](/playbooks/epub-to-audiobook.md). It runs **CPU-only** and
deliberately so: Kokoro ships no Vulkan build, its ROCm image is experimental and x86-only
against a `gfx1150` this homelab has already found flaky for ROCm, and the iGPU is
already shared between [ollama](/containers/ollama.md) and
[jellyfin](/containers/jellyfin.md). At 82M parameters it doesn't need the GPU —
measured **~4.5–6× realtime** on CPU.

The service is four lines in `/opt/stack/docker-compose.yml` with **no volume mount** —
the model weights are baked into the image, so mounting a persistent model directory over
`api/src/models` would hide them and force a re-download. The image lives on the VM's
root disk with Docker's other data (root disk went 12G → 17G of 30G on the pull).

# Storage

Two virtual disks: `virtio0` is the **root disk** on [DataPool](/storage/datapool.md)
(64G) — n8n's `/data/n8n` and Caddy's config/CA data (`/data/caddy`) live here, bind-mounted
into their containers. `virtio1` is a 200G disk on [AIVault](/storage/aivault.md),
formatted ext4 and mounted at `/mnt/aivault` — **`/data/postgres` and `/data/qdrant` are
now bind-mounts onto it** (migrated 2026-07-25; it sat attached-but-unmounted before that —
see [AIVault](/storage/aivault.md#postgresqdrant-migrated-here-2026-07-25) for the full
migration record). `docker-compose.yml` needed no changes since the bind-mount source
paths didn't move, only what backs them.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md (Dave's claude.ai account, last updated 2026-07-19)
