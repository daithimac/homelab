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

Reachable via [Caddy](/playbooks/reverse-proxy-caddy.md) at either:

* `https://sabnzbd.lan` — internal CA, browsers warn until the device trusts Caddy's root.
* `https://sabnzbd.133gsl.ie` — publicly-trusted Let's Encrypt certificate, no per-device
  trust step. See [133gsl.ie on Cloudflare DNS](/playbooks/dns-cloudflare-133gsl-ie.md).

**Every new hostname must be added to `host_whitelist` in `sabnzbd.ini` separately**, or
SABnzbd returns `403` on the proxied Host header rather than loading. `sabnzbd.lan` was
added 2026-07-25 and `sabnzbd.133gsl.ie` on 2026-07-28; the file now lists both. Restart
with **`systemctl restart sabnzbdplus@root.service`** — the unit is *not* called `sabnzbd`,
and restarting that name succeeds silently while changing nothing. Verified 2026-07-28:
`Host: sabnzbd.133gsl.ie` → `200`, an unlisted host → `403`, so the whitelist is genuinely
enforcing rather than just permissive.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md (Dave's claude.ai account, last updated 2026-07-19)
