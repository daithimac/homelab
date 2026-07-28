---
type: LXC Container
title: jellyfin (CT101)
description: Media server with hardware transcode via the iGPU, unprivileged LXC at 192.168.0.12.
tags: [proxmox, lxc, media, jellyfin]
timestamp: 2026-07-25T00:00:00Z
---

CT101, hostname `jellyfin`, unprivileged LXC, **192.168.0.12**. Media server using the
Radeon 890M iGPU for hardware transcode.

# IP — confirmed

The container's own address is **192.168.0.12**, confirmed directly in the Proxmox UI on
2026-07-25. Note `.12` still sits in the historical DHCP-collision zone (see
[IP addressing](/network/ip-addressing.md)), so re-check if it ever seems to drift.

**`jellyfin.lan` does not resolve to `.12`.** This page said so until 2026-07-28, and it
had been wrong since the [Caddy reverse proxy](/playbooks/reverse-proxy-caddy.md) was wired
up on 2026-07-25 — the rewrite was repointed at **192.168.0.14** (docker-stack, where Caddy
runs) that same day, and Caddy forwards to `.12:8096` by Host header. Verified live
2026-07-28: `dig +short jellyfin.lan @192.168.0.20` → `192.168.0.14`. Don't use the DNS
answer as a way to look up a guest's own IP; that's what
[IP addressing](/network/ip-addressing.md) is for.

# Access

* `https://jellyfin.lan` — internal CA, browsers warn until the device trusts Caddy's root.
* `https://jellyfin.133gsl.ie` — **publicly-trusted Let's Encrypt wildcard, no per-device
  trust step.** Resolves only inside the tailnet; public DNS returns nothing. Preferred.
  See [133gsl.ie on Cloudflare DNS](/playbooks/dns-cloudflare-133gsl-ie.md).

# Storage

Media library backed by the [MediaTank](/storage/mediatank.md) pool, bind-mounted.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/network-conventions.md (Dave's claude.ai account, last updated 2026-07-19)
