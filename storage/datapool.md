---
type: ZFS Pool
title: DataPool
description: 1TB Samsung SSD for VM/LXC rootfs disks and Docker volumes. Registered as Proxmox storage.
resource: /DataPool
tags: [zfs, storage]
timestamp: 2026-07-25T00:00:00Z
---

1TB Samsung SSD (928G usable). Mountpoint `/DataPool`. Registered as Proxmox ZFS storage
(disk images + container rootfs). 30.4G used / 898G free as of 2026-07-25 — plenty of
headroom.

Holds VM/LXC rootfs disks and Docker volumes, including the rootfs for
[docker-stack (VM103)](/containers/docker-stack.md). Carries more than that name suggests,
though: docker-stack's actual Postgres and Qdrant data (small — 47M and 20K respectively)
lives as plain directories on its **root disk here**, not on
[AIVault](/storage/aivault.md) as earlier docs said — see the AIVault page for the full
story and the [actions.md](/actions.md) item about the unused disk that was meant for it.

# Stray snapshot (found 2026-07-25)

`DataPool/subvol-107-disk-0@test-snap` — a manual snapshot on
[opencode (CT107)](/containers/opencode.md)'s rootfs, created 2026-07-12, small (1.46M),
not part of any documented backup routine. Nothing currently depends on it as far as this
audit could tell; low-priority cleanup candidate, not urgent.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/proxmox-storage.md (Dave's claude.ai account, last updated 2026-07-19)
