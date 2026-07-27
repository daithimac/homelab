---
type: LXC Container
title: tailscale (CT105)
description: Tailscale subnet router + exit node, unprivileged LXC at 192.168.0.16.
tags: [proxmox, lxc, tailscale, vpn, dns]
timestamp: 2026-07-19T00:00:00Z
---

CT105, hostname `tailscale`, unprivileged LXC, **192.168.0.16**. The only Tailscale node
in the house that advertises routes — the Mac Mini, MacBook Pro, and iPhone are Tailscale
*clients that roam* and never advertise routes. CT105 holds that duty because it's
always-on and on the LAN.

It runs both roles simultaneously:

* **Subnet router** — reach LAN devices/containers from anywhere without installing
  Tailscale on each one. This is the one Dave usually actually wants.
* **Exit node** — route all internet traffic through the house. Kept available but only
  *enabled* on a client from dodgy hotel Wi-Fi or when an Irish IP is wanted abroad —
  daily use would route every stream through the home upload.

Full setup (tun passthrough, IP forwarding, route/exit-node approval, GRO forwarding
persistence) is in [Tailscale subnet router & exit node](/playbooks/tailscale-subnet-router-exit-node.md).

# DNS dependency

All personal/tailnet devices get DNS from [adguard (CT109)](adguard.md), delivered over
Tailscale via this container's subnet route. If CT105 or CT109 is down, tailnet devices
lose DNS until Tailscale is toggled off on the client. Details:
[DNS via AdGuard](/network/dns-adguard.md).

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/tailscale.md (Dave's claude.ai account, last updated 2026-07-19)
