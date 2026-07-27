---
type: LXC Container
title: sabnzbd (CT108)
description: SABnzbd Usenet downloader onto MediaTank, LXC at 192.168.0.19.
tags: [proxmox, lxc, sabnzbd, media, avahi]
timestamp: 2026-07-25T00:00:00Z
---

CT108, hostname `sabnzbd`, **192.168.0.19**. Runs SABnzbd for Usenet downloads, writing
onto the [MediaTank](/storage/mediatank.md) pool.

# Also runs avahi-daemon (undocumented until 2026-07-25)

Broadcasts mDNS (`sabnzbd.local`) — found during a fleet-wide audit, no other guest runs
it. `/etc/avahi/services/` is empty (no custom service records, just default hostname
announcement), and it isn't in any other container's package set, so this looks like a
side effect of however SABnzbd was originally installed here rather than a deliberate
choice. Benign as far as it's been checked, but worth knowing it's the one guest visible
over mDNS — see the `.local` vs `.lan` boundary note in
[DNS via AdGuard](/network/dns-adguard.md).

# Reverse-proxied

Reachable as `https://sabnzbd.lan` via [Caddy](/playbooks/reverse-proxy-caddy.md) — needed
`sabnzbd.lan` added to `host_whitelist` in `sabnzbd.ini` to stop it 403ing on the proxied
Host header (see the playbook's gotchas section).

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md (Dave's claude.ai account, last updated 2026-07-19)
