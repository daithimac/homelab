---
type: Reference
title: IP addressing
description: Subnet, gateway, the static IP ledger for every guest, and the open DHCP-conflict problem.
tags: [network, ip, dhcp]
timestamp: 2026-07-25T00:00:00Z
---

# Subnet & addressing

* Subnet **192.168.0.x/24**, gateway `192.168.0.1`. NOT 192.168.1.x — a mismatched
  gateway on that wrong subnet locked Dave out of the Proxmox GUI once.
* New guests should be built to policy from the start rather than fixed afterwards —
  [audiobooks (CT111)](/containers/audiobooks.md) (2026-07-26) was the first created after
  the fleet-wide cleanup below and got its static IP, AdGuard DNS and IPv6-off sysctl at
  provisioning time.
* Guests are meant to use **static** IPs, **AdGuard for DNS**
  ([policy changed 2026-07-25](/network/dns-adguard.md#guest-dns-policy--changed-2026-07-25)),
  and **IPv6 off**. As of 2026-07-25 this is now true fleet-wide — see below.

# IPv6 — off fleet-wide as of 2026-07-25

A same-day review of the live Proxmox UI first found **global IPv6 addresses
(`2a02:8084:...`)** assigned on 9 of 12 guests (nas, jellyfin, ollama, docker-stack,
opencode, sabnzbd, adguard, grafana, sillytavern), despite that contradicting documented
policy. Root cause: only [openwebui (CT106)](/containers/openwebui.md) had ever had IPv6
explicitly disabled, via `/etc/sysctl.d/99-disable-ipv6.conf` (`net.ipv6.conf.*.disable_ipv6
= 1`), set by the Proxmox community-scripts installer at creation time — nothing else in
the fleet had it. [tailscale (CT105)](/containers/tailscale.md) never had global IPv6 for
unrelated reasons (its own network manager). Rather than assume, this was raised as a
decision (fix vs. drop the policy) — Dave chose to fix it fleet-wide. Replicated openwebui's
sysctl file on the other 8 LXCs and on docker-stack (VM, via `qm guest exec`); applied
**live** in all cases (`sysctl -p`, no reboot needed — global addresses drop immediately).
[home-assistant (VM104)](/containers/home-assistant.md) had already been fixed earlier the
same day via `ha network update ... --ipv6-method disabled`. All 12 guests now match policy.

# Static IP ledger

| IP            | Host                                                     | Verified? |
|---------------|-----------------------------------------------------------|-----------|
| 192.168.0.11  | [nas (CT100)](/containers/nas.md)                          | yes |
| 192.168.0.12  | [jellyfin (CT101)](/containers/jellyfin.md)                | yes — confirmed in Proxmox UI 2026-07-25 (previously only inferred from AdGuard rewrites) |
| 192.168.0.13  | [ollama (CT102)](/containers/ollama.md)                    | yes |
| 192.168.0.14  | [docker-stack (VM103)](/containers/docker-stack.md)        | yes — **confirmed static** in the guest's own netplan config, 2026-07-25 (hardcoded address, no `dhcp4`); DNS fixed to point at AdGuard the same day — see that page |
| 192.168.0.16  | [tailscale (CT105)](/containers/tailscale.md)               | yes |
| 192.168.0.17  | [openwebui (CT106)](/containers/openwebui.md)               | yes |
| 192.168.0.18  | [opencode (CT107)](/containers/opencode.md)                 | yes |
| 192.168.0.19  | [sabnzbd (CT108)](/containers/sabnzbd.md)                   | yes |
| 192.168.0.20  | [adguard (CT109)](/containers/adguard.md)                   | yes — confirmed static in `pct config 109` |
| 192.168.0.21  | [grafana (CT110)](/containers/grafana.md)                   | yes — discovered in Proxmox UI 2026-07-25, was missing from this documentation entirely |
| 192.168.0.22  | [sillytavern (CT120)](/containers/sillytavern.md)            | yes — was DHCP (confirmed via `pct config 120`), **converted to static same day** via `pct set 120 -net0 ...`, DNS also switched to AdGuard; see that page |
| 192.168.0.23  | [home-assistant (VM104)](/containers/home-assistant.md)    | yes — was a DHCP lease at `.213` (confirmed via `ha> network info`, 2026-07-25), **moved to static `.23` same day** via `ha network update`; IPv6 disabled and DNS pointed at AdGuard to match the fleet — see that page |
| 192.168.0.24  | [audiobooks (CT111)](/containers/audiobooks.md)             | yes — assigned at creation 2026-07-26; `.24` confirmed unanswered by `ping` from both the host and Dave's Mac before allocating, and it sits above the historical `.12/.14/.16` collision zone |
| 192.168.0.25  | **UNKNOWN DEVICE** — MAC `5c:1b:f4:88:46:08`               | occupied, unidentified. Found answering `ping` on 2026-07-26 while allocating for CT112. Not a documented guest. See [actions.md](/actions.md) |
| 192.168.0.26  | [audiobookshelf (CT112)](/containers/audiobookshelf.md)     | yes — assigned at creation 2026-07-26 after `.25` came back occupied; `.26` confirmed silent by `ping` from the host before allocating |
| 192.168.0.27  | [code-server (CT113)](/containers/code-server.md)           | yes — assigned at creation 2026-07-26; `.27` (and `.28`) re-confirmed silent by `ping` from the host at allocation time, not taken on the earlier sweep's word |

Free below `.27`: **`.15` only** — the single gap in the otherwise contiguous static block.
It's usable, but it sits in the historical `.12`/`.14`/`.16` collision zone, which is why
CT111, CT112 and CT113 all built upward instead. `.28`–`.32` were confirmed free on
2026-07-26.

When adding a device, pick the next free static and check it's not in the DHCP range.
**Actually run the ping** — this is not a formality. `.25` was the obvious next address for
CT112 and it was already taken by something nobody has identified; assuming it was free
would have produced an IP conflict on a live build.

# The DHCP-conflict problem (open)

Static assignments have repeatedly collided with the router's DHCP pool (.12/.14/.16
churn during Open WebUI setup). Standing recommendation, **not yet actioned**: shrink the
router DHCP pool to `.100–.200` and keep an IP ledger. Until then, assign new statics
below .100 and verify the address is actually free first (`ping`, or check the router
client list).

`home-assistant` (previously at .213) was **confirmed** DHCP fallout — its HAOS interface
was left on `method: auto` — and has since been moved to static `.23` (2026-07-25, see
[home-assistant (VM104)](/containers/home-assistant.md)). `docker-stack`'s `.14` was
checked too and is **confirmed static** in the guest's netplan (2026-07-25) — it just
happens to sit in the historical collision zone, not an active conflict.

# Citations

[1] n5-pro-homelab Claude Skill — references/network-conventions.md (Dave's claude.ai account, last updated 2026-07-19)
[2] Direct Proxmox UI review (Dave's homelab, 2026-07-25) — jellyfin/docker-stack/home-assistant IPs, IPv6 status, and the grafana/sillytavern discovery.
