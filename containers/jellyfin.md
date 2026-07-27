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

AdGuard's DNS rewrites point `jellyfin.lan` at **192.168.0.12**. Previously unverified
against the container's own config; confirmed directly in the Proxmox UI on 2026-07-25.
Note `.12` still sits in the historical DHCP-collision zone (see
[IP addressing](/network/ip-addressing.md)), so re-check if it ever seems to drift.

# Storage

Media library backed by the [MediaTank](/storage/mediatank.md) pool, bind-mounted.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/network-conventions.md (Dave's claude.ai account, last updated 2026-07-19)
