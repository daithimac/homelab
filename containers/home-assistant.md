---
type: VM
title: home-assistant (VM104)
description: Home Assistant OS VM for smart-home duty, at 192.168.0.23 (static).
tags: [proxmox, vm, home-assistant, smart-home]
timestamp: 2026-07-25T00:00:00Z
---

VM104, running **Home Assistant OS (HAOS)**. Handles smart-home duty for the household.
Sits on the same `192.168.0.x/24` as every other guest — see
[IP addressing](/network/ip-addressing.md).

**No Supervisor add-ons installed** (confirmed via `ha addons list` → `addons: []`,
2026-07-25 fleet audit) — this is core HA only, no ESPHome/Node-RED/Mosquitto/etc.
running alongside it.

# IP — 192.168.0.23, static (fixed 2026-07-25)

Originally found on DHCP at **192.168.0.213** (confirmed via `ha> network info` on
2026-07-25), breaking the fleet's "static IPs, IPv6 off, DNS via AdGuard" convention. The
`enp6s18` interface had been left on `method: auto` since setup — a HAOS VM's networking
is configured from *inside* the guest, so Proxmox's own static-IP convention for LXCs
never applied to it.

**Fixed same day**, from the HAOS CLI (`ha>` prompt in the Proxmox console, no login
required):

```
network update enp6s18 --ipv4-method static --ipv4-address 192.168.0.23/24 \
  --ipv4-gateway 192.168.0.1 --ipv4-nameserver 192.168.0.20 --ipv6-method disabled
```

`.23` was picked as the next free slot after sillytavern (`.22`) and confirmed unused
first (`ping` from the host, 100% loss before the change). Verified after: `ha> network
info` shows `method: static` on `192.168.0.23/24`, IPv6 `method: disabled` (link-local
only remains), nameserver `192.168.0.20` (AdGuard). Confirmed reachable at `.23` and the
old `.213` lease gone (`ping` from the host, 0% vs 100% loss respectively).

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md (Dave's claude.ai account, last updated 2026-07-19)
[2] `ha> network info` output and `ha> network update` fix, HAOS console via Proxmox (2026-07-25)
