---
type: LXC Container
title: sillytavern (CT120)
description: SillyTavern chat frontend against Ollama, unprivileged LXC at 192.168.0.22.
tags: [proxmox, lxc, sillytavern, ai, ollama]
timestamp: 2026-07-25T00:00:00Z
---

CT120, hostname `sillytavern`, unprivileged LXC, static **192.168.0.22**. Chat frontend for
[ollama (CT102)](ollama.md), running natively via Node.js — not Docker (`features:
nesting=0` in the LXC config rules out a nested container runtime).

# IP was DHCP until 2026-07-25 — now fixed

Originally found on DHCP (`net0: ...,ip=dhcp` in `pct config 120`), unlike every other
guest's hardcoded address — the same class of risk that bit
[home-assistant (VM104)](/containers/home-assistant.md). Its DNS also pointed at the router
(`192.168.0.1`) instead of AdGuard. Both fixed the same day at the **Proxmox level** (not
inside the guest, since LXC networking is host-managed): `pct set 120 -net0
name=eth0,bridge=vmbr0,hwaddr=BC:24:11:4A:51:E9,ip=192.168.0.22/24,gw=192.168.0.1,type=veth
-nameserver 192.168.0.20`, then `pct reboot 120`. Verified after reboot: `ip -4 addr show
eth0` shows the static `.22/24` with no DHCP lease markers, `/etc/resolv.conf` shows
`nameserver 192.168.0.20`, and the SillyTavern web UI answers `200` on port 8000.

# Backend

Runs as systemd unit `sillytavern.service`: dedicated `sillytavern` user,
`WorkingDirectory=/opt/sillytavern/app`, `ExecStart=npm run start`,
`NODE_ENV=production`, Node v22.23.1, listening on `0.0.0.0:8000`.

Connects to [ollama (CT102)](ollama.md) directly over the network
(`http://192.168.0.13:11434`, `main_api: textgenerationwebui`) rather than through
[openwebui (CT106)](openwebui.md). Configured models (from
`data/default-user/settings.json`):

* Chat: `hf.co/mradermacher/Moonlit-Mirage-12B-i1-GGUF:latest`
* Embeddings: `mxbai-embed-large`

# Why CT120, not CT111

The ID jumps from CT109/CT110 straight to CT120, unlike every other guest's sequential
numbering. No leftover ZFS or LVM volumes exist for IDs 111–119 on this host, so there's no
evidence a container in that range was created and destroyed — the gap looks like a
deliberate reserved block rather than reused space, but the reason is still unconfirmed.

# Citations

[1] Live `pct exec 120` shell review via the Proxmox web console (2026-07-25).
