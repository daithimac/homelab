---
type: Reference
title: Thunderbolt link (host-level, working)
description: The TB4 link between the N5 Pro and the Mac — dead since install, fixed 2026-07-26, now carrying ~300MB/s host-to-Mac.
tags: [network, thunderbolt]
timestamp: 2026-07-26T00:00:00Z
---

TB4 between [the N5 Pro host](/host/n5-pro.md) (`10.10.10.2`) and the Mac mini
(`10.10.10.1`), on a **`/30`** — only those two addresses exist in the subnet, there is no
room for a third host on it.

**The link was non-functional from the Proxmox install until 2026-07-26.** It is up and
verified now; the sections below record why it was dead, because the failure mode is
subtle and the previous docs asserted the opposite of the live state.

# The naming bug that kept it down

The interface is **renamed to `nic0` by a systemd `.link` file**, not by a udev rule:
`/usr/local/lib/systemd/network/50-pmx-nic0.link`, written by the Proxmox installer
alongside `50-pmx-nic1.link` / `50-pmx-nic2.link` for the two onboard Realtek NICs. The
installer pins each name to a **MAC address**:

```
[Match]
MACAddress=02:a4:d5:51:b2:25    # the TB netdev's MAC on install day, 2026-07-04
Type=ether
[Link]
Name=nic0
```

That match stopped being true. `thunderbolt-net`'s MAC is locally administered and tied to
the current host↔peer pairing (live value `02:9b:2f:75:1f:55` — stable across an interface
recreate, but not equal to the installer's value). With the match failing, **nothing
renamed anything**: the interface sat there as `thunderbolt0`, `state DOWN`, no address.

Two things then silently did nothing, each of which looks correct in isolation:

* `/etc/network/interfaces` carries `auto nic0` / `iface nic0 inet static` with
  `address 10.10.10.2/30` — a stanza for an interface that did not exist.
* `/etc/udev/rules.d/99-tb-net.rules` matched `KERNEL=="nic0"`. `KERNEL` is the *kernel*
  name (`thunderbolt0`); the renamed name never appears there, so the rule could not have
  fired even if the rename had worked.

The `nic0 not recognized` warning seen during `ifreload -a` on 2026-07-25 was **this**, not
the benign hotplug behaviour it was logged as at the time.

The cable and the pairing were never the problem — `dmesg` had been reporting the peer
correctly the whole time (`thunderbolt 0-2: new host found, vendor=0xa27`,
`thunderbolt 0-2: Apple Inc. Mac14,3`), and `/sys/bus/thunderbolt/devices/0-2` shows the
Mac.

# The fix (2026-07-26)

Match on the **driver** instead of the MAC, so the rename survives any MAC change:

```
# /usr/local/lib/systemd/network/50-pmx-nic0.link   (orig kept as .bak-20260726)
[Match]
Driver=thunderbolt-net
[Link]
Name=nic0
```

and trigger `ifup` from the renamed-device event, off the same driver property
(`.bak-20260726` kept here too):

```
# /etc/udev/rules.d/99-tb-net.rules
ACTION=="add", SUBSYSTEM=="net", ENV{ID_NET_DRIVER}=="thunderbolt-net", \
  RUN+="/usr/bin/systemd-run --no-block /usr/sbin/ifup nic0"
```

`systemd-run --no-block` matters: `RUN+=` must not block the udev event, and a bare
`ifup` there is long-running.

`thunderbolt-net` still destroys and recreates the netdev on every cable event, so the
udev trigger remains necessary — `allow-hotplug` alone does not cover it.

**Verified after the change**, rather than assumed: `modprobe -r thunderbolt_net` then
`modprobe thunderbolt_net` (a stand-in for a replug) moved the interface's ifindex 66 → 67,
proving a genuine destroy/recreate, and it came back **`nic0`, UP, with `10.10.10.2/30`
already applied**. Ping across the link is ~0.5ms, 0% loss.

The Mac side needed no change: **Thunderbolt Bridge** (`bridge0`, members `en2`/`en3`) was
already configured manually with `10.10.10.1 / 255.255.255.252` and flipped from
`status: inactive` to `status: active` on its own the moment the host brought its end up.

# What it's good for

Measured 2026-07-26, MTU 1500, default offloads (no `ethtool -K` tuning needed):

| Path | Throughput |
| --- | --- |
| host → Mac, 2GB over SSH, via TB | **311 MB/s** |
| host → Mac, 2GB over SSH, via Ethernet | 116 MB/s (1GbE, saturated) |
| Mac → host, 4GB raw TCP (`nc`), via TB | ~281 MB/s (`nc` CPU-bound at 90%) |

So ~2.7× Ethernet, and both TB figures are bounded by SSH crypto and `nc`, not by the
link — the real ceiling is higher and untested. **The old 85KB-stall symptom did not
reproduce**, in either direction, with checksum/TSO/GSO/GRO all left on.

SSH to the host over TB works (`ssh root@10.10.10.2`); the host key is the same one as
`192.168.0.10` (`SHA256:cCuq9qv8s/cGKc+wfB76VGHwZTEwUnX/Lch+oncSGv4`) and is now pinned for
both addresses in the Mac's `known_hosts`.

MTU is 1500 on both ends and untuned; the Linux side reports `maxmtu 65522`. Raising it is
the obvious next lever and needs `sudo` on the Mac — see [actions.md](/actions.md).

# SMB still goes over Ethernet

**SMB to the Mac still goes over standard Ethernet to 192.168.0.11
([nas / CT100](/containers/nas.md)), NOT over Thunderbolt.** The TB link reaches the
*host*, and [nas](/containers/nas.md) is an unprivileged LXC with no interface on it — the
`/30` has no spare address for one, so putting the container on TB means re-addressing the
link and bridging or routing into the guest.

That is exactly what the abandoned NAT-over-TB experiment tried. Its leftovers (a
`bridge-ports none` `vmbr1`, plus a `10.10.10.4/24` `net1` on nas) were removed 2026-07-25
— and worth noting in hindsight: `vmbr1` had **no uplink because the interface it was
meant to enslave, `nic0`, did not exist**. The same naming bug that kept the link down is
what left that bridge dangling. Whether to retry SMB over TB now that the link demonstrably
works is an open decision in [actions.md](/actions.md), not a settled "no".

For stable SMB from the Mac, use the **Tailscale MagicDNS hostname** (or a router DHCP
reservation), not a raw IP that can change.

# Citations

[1] n5-pro-homelab Claude Skill — references/network-conventions.md (Dave's claude.ai account, last updated 2026-07-19)
[2] Direct host/Mac console review, 2026-07-26 — `systemd.link` and udev diagnosis, fix, and throughput measurements.
