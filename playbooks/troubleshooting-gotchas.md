---
type: Playbook
title: Recurring gotchas
description: Known-bitten issues on the N5 Pro host — check these first when something breaks.
tags: [troubleshooting, gotchas]
timestamp: 2026-07-19T00:00:00Z
---

These have all bitten before on [the N5 Pro host](/host/n5-pro.md). When a symptom
matches, reach here before first principles.

* **iGPU is `card1` / `226:1`, not `card0`/`226:0`.** The render node is `renderD128`
  (`226:128`). See [GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md).
* **Unprivileged LXC device access** shows the device owned by `65534:65534` (nobody)
  inside the container until an idmap maps the group through. See
  [GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md) for the exact fix.
* **Bind-mount chown must use the mapped UID from the host**, not `1000` and not from
  inside the container. Unprivileged root maps to host **100000**. So
  `chown -R 100000:100000 /AIVault/ollama-models` **on the host**, never `chown 1000`
  inside the container. See [ZFS pools & bind-mount permissions](zfs-bind-mount-permissions.md).
* **IPv6 mirror failures on apt** in every LXC → add `Acquire::ForceIPv4 "true"` or run
  `apt -o Acquire::ForceIPv4=true update`.
* **Debian Bookworm base ships Mesa 22.3.6**, too old for the RDNA 3.5 iGPU. Needs
  bookworm-backports Mesa (25.x) for RADV to see the card. See
  [GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md).
* **IP conflicts from DHCP overlap.** Static assignments have collided with the DHCP pool
  repeatedly. Standing recommendation (not yet actioned): shrink router DHCP pool to
  `.100–.200` and keep an IP ledger. See [IP addressing](/network/ip-addressing.md).
* **"All websites failing" on tailnet devices = check [adguard (CT109)](/containers/adguard.md)
  first.** All personal-device DNS is hard-wired to 192.168.0.20 via Tailscale with no
  fallback. Intermittent timeouts with some domains resolving (cache hits) means AdGuard's
  *upstream* is flaky, not AdGuard down — this bit once with a dead Quad9 DoH endpoint as
  the sole upstream. Debug: `dig @192.168.0.20 <domain>`, then
  `pct exec 109 -- journalctl -u AdGuardHome | grep "exchange fa"`.
* **Full-fleet shutdown/reboot kills LAN DNS (AdGuard, CT109)** — remote sessions and
  anything resolving names will fail mid-maintenance; sequence AdGuard last down / first
  up. Details:
  [gpu-passthrough-ollama-vulkan.md](gpu-passthrough-ollama-vulkan.md#uma-and-gtt--the-memory-pool-and-how-its-split).
* **A `.lan` domain 403s instead of loading in the browser.** Check the backend app for its
  own Host-header whitelist (e.g. SABnzbd's `host_whitelist` in `sabnzbd.ini`) — arriving
  via [Caddy](/playbooks/reverse-proxy-caddy.md) changes the `Host:` header to the `.lan`
  name, which some apps reject unless that exact name is allow-listed.
* **If the Thunderbolt link is dead, check the interface's *name* before anything else.**
  It is renamed to `nic0` by `/usr/local/lib/systemd/network/50-pmx-nic0.link`, and the
  Proxmox installer wrote that file with a **`MACAddress=` match** — which went stale and
  left the interface sitting as `thunderbolt0`, DOWN, for the entire life of this install
  until 2026-07-26. When that happens the `auto nic0` stanza in `/etc/network/interfaces`
  and the udev `ifup` rule both silently apply to nothing, and `ifreload -a` says only
  `nic0 not recognized`. Now matched on `Driver=thunderbolt-net` instead. Check with
  `ip -br addr | grep -e nic0 -e thunderbolt` — seeing `thunderbolt0` at all means the
  rename failed. The TB link (`10.10.10.2/30`, Mac at `10.10.10.1`) is **host-level** (ZFS
  sends, VM disk transfers, ~300MB/s vs 116MB/s over Ethernet); SMB to the Mac goes over
  **standard Ethernet to 192.168.0.11**. See
  [Thunderbolt link](/network/thunderbolt-link.md).

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md (Dave's claude.ai account, last updated 2026-07-19)
