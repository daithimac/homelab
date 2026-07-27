---
type: LXC Container
title: nas (CT100)
description: Samba / NAS container, unprivileged LXC at 192.168.0.11.
tags: [proxmox, lxc, nas, samba]
timestamp: 2026-07-25T00:00:00Z
---

CT100, hostname `nas`, unprivileged LXC at **192.168.0.11**. Runs Samba, serving as the
NAS front-end for the homelab.

# Storage

Backed by the [MediaTank](/storage/mediatank.md) pool, bind-mounted (not a Proxmox-managed
disk).

# Notes

SMB access from the Mac goes over **standard Ethernet to 192.168.0.11**, not the
Thunderbolt link — see [Thunderbolt link](/network/thunderbolt-link.md), where the TB/NAT
path was tried and abandoned. For a stable SMB target from the Mac, use the Tailscale
MagicDNS hostname or a router DHCP reservation rather than a raw IP.

**The TB link itself was repaired 2026-07-26** and now carries ~300MB/s host↔Mac, but it
terminates on the *host*: this container has no interface on it, and the link's `/30`
holds no spare address for one. Moving SMB onto TB is therefore a real re-design (re-address
the link, then bridge or route into the guest), logged as an open decision in
[actions.md](/actions.md) rather than done unprompted.

**Had a second, dead NIC left over from that abandoned attempt** — `net1`, `10.10.10.4/24`
on an isolated bridge `vmbr1` with no uplink to anything. Found in a 2026-07-25 network
audit, confirmed genuinely dead (290B rx / 11KB tx total, just ARP noise), and **removed
the same day**: `pct set 100 -delete net1`, then the `vmbr1` stanza dropped from
`/etc/network/interfaces` (backed up first to `.bak-20260725`) and applied live with
`ifreload -a`. Verified after: `vmbr1` no longer exists on the host, no `eth1` inside the
container, main connectivity (`vmbr0`, DNS) unaffected. In hindsight that bridge had no
uplink for a concrete reason: the interface it was meant to enslave, `nic0`, did not exist
at the time — the same naming bug that kept the whole TB link down.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/network-conventions.md (Dave's claude.ai account, last updated 2026-07-19)
