---
type: LXC Container
title: adguard (CT109)
description: AdGuard Home — tailnet-wide DNS and ad blocking, unprivileged LXC at 192.168.0.20.
tags: [proxmox, lxc, adguard, dns]
timestamp: 2026-07-19T00:00:00Z
---

CT109, hostname `adguard`, unprivileged LXC, **192.168.0.20** (confirmed static in
`pct config 109`). Runs AdGuard Home as the global nameserver for every tailnet/personal
device in the house, delivered over Tailscale via [tailscale (CT105)](tailscale.md)'s
subnet route.

**"All websites failing" on tailnet devices → check this container first.** Every
personal-device DNS query is hard-wired to 192.168.0.20 with no fallback. Intermittent
timeouts with some domains resolving (cache hits) usually mean AdGuard's *upstream* is
flaky, not AdGuard itself being down — this bit once with a dead Quad9 DoH endpoint as the
sole upstream. Debug:

```bash
dig @192.168.0.20 <domain>
pct exec 109 -- journalctl -u AdGuardHome | grep "exchange fa"
```

Full upstream config, rewrites, and design rationale:
[DNS via AdGuard](/network/dns-adguard.md).

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/network-conventions.md (Dave's claude.ai account, last updated 2026-07-19)
