---
type: Service
title: Minisforum N5 Pro (Proxmox host)
description: Single Proxmox host running NAS, AI inference, database, media, and smart-home duty for Dave's homelab.
tags: [proxmox, host, homelab]
timestamp: 2026-07-25T00:00:00Z
---

Dave runs a **Minisforum N5 Pro** (Ryzen AI, Radeon 890M iGPU, 96GB RAM, 128GB NVMe boot
disk) as a single Proxmox host. It carries NAS, AI inference, database, media, and
smart-home duty simultaneously — there is no fleet, no second host, and no HA cluster.
Commands and topology should always be proposed for *this* machine, not generic Proxmox
advice.

# Network

Subnet is **192.168.0.x/24** (NOT 192.168.1.x — that mismatch locked Dave out of the
Proxmox GUI once via a bad gateway). Gateway `192.168.0.1`. IPv6 is off on all guests as of
2026-07-25 — a same-day review found it wasn't actually enforced fleet-wide (only openwebui
had it disabled), so it was rolled out to the rest; see
[IP addressing](/network/ip-addressing.md).

**No Proxmox firewall is actually active anywhere** (confirmed 2026-07-25): no
`datacenter.cfg` firewall enable, no `cluster.fw`, no `host.fw`, and `/etc/pve/firewall/`
is empty — zero rule files for any guest. Four guests (nas, jellyfin, ollama, tailscale)
have `firewall=1` set on their `net0`, but that flag is a no-op without the subsystem
enabled; security here is entirely the router/LAN boundary, not Proxmox. Not treated as a
bug — just made explicit since the `firewall=1` flags could otherwise read as false
assurance.

# SSH access

`ssh root@192.168.0.10` from Dave's Mac, using the default `~/.ssh/id_ed25519` — no
`~/.ssh/config` entry needed, and no agent. Fixed 2026-07-26; before that the key wasn't
installed host-side and every session had to fall back to the web console.

**Authorized keys are cluster-managed, not a plain file.** `/root/.ssh/authorized_keys` is
a symlink to **`/etc/pve/priv/authorized_keys`** on the pmxcfs FUSE mount — edit that path,
and note it's replicated by Proxmox rather than being ordinary local state. `sshd` is
otherwise stock: `permitrootlogin yes`, `pubkeyauthentication yes`.

When adding a key, **verify the result by fingerprint** (`ssh-keygen -lf
/etc/pve/priv/authorized_keys`) rather than reading the base64 back — especially when
typing through the web console, where a single mangled character produces a line that
looks plausible and silently never authenticates.

Two ED25519 keys in that file commented `ward@Davids-Mac-mini` match **no key currently on
the Mac** — see [actions.md](/actions.md) before assuming they're yours.

# What runs here

Fifteen LXC containers and VMs — see [Containers & VMs](/containers/index.md) for the full
list (two of these, grafana and sillytavern, were found running but undocumented as of
2026-07-25; [audiobooks (CT111)](/containers/audiobooks.md),
[audiobookshelf (CT112)](/containers/audiobookshelf.md) and
[code-server (CT113)](/containers/code-server.md) were all added 2026-07-26). Three
ZFS pools imported from a retired TrueNAS install — see [Storage](/storage/index.md).

# Working style with Dave

* Dave is a **cloud engineer (Google Cloud PSO)** — fluent in infra concepts, so skip
  101-level explanation of what a container or a systemd unit *is*. He's newer to home
  networking and NAS specifics than to cloud, so don't assume deep prior knowledge there.
* During active troubleshooting: **commands only, no preamble.** Lead with the command;
  explain only if asked or if a step is genuinely dangerous.
* **Verify state before asserting it** — `systemctl show`, `cat` the real config file,
  `ls -ln` the actual device. Several dead ends in past sessions came from trusting an
  editor view or an assumed default instead of checking the live value.
* When a change touches an unprivileged LXC, flag any bind mount whose ownership could
  shift under the idmap before rebooting.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md (Dave's claude.ai account, last updated 2026-07-19)
