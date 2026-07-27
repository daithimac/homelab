---
type: Playbook
title: Tailscale subnet router & exit node (CT105)
description: Setup for tun passthrough, IP forwarding, route/exit-node advertisement and approval, and GRO forwarding persistence on CT105.
tags: [tailscale, vpn, lxc, proxmox, dns]
timestamp: 2026-07-19T00:00:00Z
---

Tailscale runs in [tailscale (CT105)](/containers/tailscale.md), unprivileged LXC,
192.168.0.16. Other Tailscale nodes (Mac Mini, MacBook Pro, iPhone, and — confirmed via
`tailscale status` 2026-07-25, not previously listed — an iPad) are clients that roam and
never advertise routes — all routing/exit-node duty lives on CT105 because it's always-on
and on the LAN.

# The distinction Dave conflates (state it every time)

* **Subnet router** = reach LAN devices/containers from anywhere without installing
  Tailscale in each one. This is the thing he usually actually wants. Advertise
  `192.168.0.0/24`.
* **Exit node** = route ALL internet traffic through the house. Advertise it (free to
  have), but only *enable* it on a client from dodgy hotel Wi-Fi or when wanting an Irish
  IP abroad. Daily use routes every stream through the home upload — a garden hose in
  Ireland.

# Setup (CT105, unprivileged — needs tun passthrough)

### 1. `/dev/net/tun` passthrough (host: `/etc/pve/lxc/105.conf`)
Check first inside CT105: `ls -l /dev/net/tun`. If missing, on the host add:
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
`pct stop 105 && pct start 105`.

### 2. IP forwarding (inside CT105)
```bash
echo 'net.ipv4.ip_forward=1' | tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding=1' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### 3. Advertise both (inside CT105)
```bash
tailscale up --advertise-routes=192.168.0.0/24 --advertise-exit-node --accept-routes
```

### 4. Approve in the admin console
Machines → CT105 node → `···` → Edit route settings → tick BOTH the
`192.168.0.0/24` subnet route AND "Use as exit node". Nothing works until approved here —
the step people forget.

### 5. Clients
* Subnet routes: enable "Accept routes" (iOS: app settings; macOS: `--accept-routes` or
  menu).
* Exit node: toggle ON only when needed, OFF for daily use.

# GRO forwarding optimisation

`tailscale up` warns: `UDP GRO forwarding is suboptimally configured on eth0`. Not an
error — a throughput optimisation for a forwarding node. Fix (inside CT105):
```bash
apt install -y ethtool
ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
ethtool -k eth0 | grep -E 'rx-udp-gro-forwarding|rx-gro-list'   # want: on / off
```
Applied cleanly inside the unprivileged container here (the veth-offload permission
caveat didn't bite). Persist across reboots with a oneshot service:
```bash
cat > /etc/systemd/system/tailscale-gro.service <<'EOF'
[Unit]
Description=Set ethtool GRO for Tailscale forwarding
After=network-online.target
Wants=network-online.target
[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
[Install]
WantedBy=multi-user.target
EOF
systemctl enable --now tailscale-gro.service
```
(`which ethtool` to confirm the `/usr/sbin/ethtool` path matches; a oneshot showing
`inactive (dead)` with `status=0/SUCCESS` is correct, not broken.)

# The real finish line

The config is theoretical until proven from a remote client. Test from iPhone on
**cellular** (Wi-Fi off), "Accept routes" on: load a container by LAN address, e.g.
`http://192.168.0.17:8080` ([openwebui](/containers/openwebui.md)) or ping
`192.168.0.13` ([ollama](/containers/ollama.md)). Then toggle CT105 as exit node, check a
"what's my IP" site shows the Irish home IP, toggle back off. If subnet access misbehaves,
the self-referential route (CT105 is on the same /24 it advertises) is the first suspect,
then Tailscale ACLs.

# DNS (AdGuard over Tailscale — replaced NextDNS, Jul 2026)

Full upstream config and rewrites: [DNS via AdGuard](/network/dns-adguard.md). In brief:
admin console → DNS → custom nameserver `192.168.0.20`, **Override DNS servers** ON,
**Use with exit node** ON (without the latter, enabling the exit node silently swaps DNS
away from AdGuard). "Restrict to domain" stays OFF — that's split DNS and would kill
global filtering. Reachability relies on CT105's subnet route, not the exit node.

Consequences to remember:
* Every tailnet device's DNS depends on [adguard (CT109)](/containers/adguard.md) **and**
  CT105 being up. Homelab down = personal devices lose DNS until Tailscale is toggled off
  on the client.
* macOS "Connect on Demand" makes Tailscale re-enable itself; the app also re-asserts the
  profile. Dave chose to leave it on — desirable now that DNS lives behind the tailnet.
* Exit node use also gets AdGuard filtering (thanks to Use-with-exit-node), but routes all
  traffic through the home connection: counts down+up against Virgin, throttled by home
  upload. Exit node = dodgy-wifi tool, not a daily default.

# Citations

[1] n5-pro-homelab Claude Skill — references/tailscale.md (Dave's claude.ai account, last updated 2026-07-19)
