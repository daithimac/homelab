---
okf_version: "0.1"
---

# Dave's Homelab

Knowledge bundle for Dave's homelab: a single Minisforum N5 Pro running Proxmox, doing
NAS, AI inference, database, media, and smart-home duty. Source knowledge was extracted
from the `n5-pro-homelab` Claude Skill (working knowledge accumulated across many
troubleshooting sessions) on 2026-07-24.

# Host

* [Minisforum N5 Pro](host/n5-pro.md) - the physical host: hardware, subnet, working style notes.

# Containers & VMs

* [Containers & VMs](containers/index.md) - all 14 LXC containers and VMs running on the host.

# Storage

* [Storage](storage/index.md) - the three ZFS pools and how they're used.

# Playbooks

* [Playbooks](playbooks/index.md) - step-by-step procedures for GPU/Ollama, Tailscale, ZFS
  permissions, audiobook generation, and the recurring gotchas to check first.

# Network

* [Network](network/index.md) - IP addressing scheme, DNS/AdGuard setup, and the
  Thunderbolt link's limited role.

# Open Actions

* [Open Actions](actions.md) - prioritised list of confirmed issues and follow-ups not yet actioned.
