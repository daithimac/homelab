---
type: Playbook
title: ZFS pools & bind-mount permissions
description: Health checks, reboot survival, and the unprivileged-LXC UID-mapping dance for bind mounts.
tags: [zfs, storage, lxc, proxmox, permissions]
timestamp: 2026-07-19T00:00:00Z
---

Applies to all three pools — [MediaTank](/storage/mediatank.md),
[AIVault](/storage/aivault.md), [DataPool](/storage/datapool.md) — imported from a retired
TrueNAS install (plain ZFS; Proxmox reads them natively). Mountpoints were normalised off
TrueNAS's `/mnt/` prefix to root paths. [DataPool](/storage/datapool.md) and
[AIVault](/storage/aivault.md) are registered as Proxmox ZFS storage (disk image +
container); [MediaTank](/storage/mediatank.md) stays unregistered — bind-mounted into
containers, never carved into VM disks.

# Health / mountpoint / import sanity

```bash
zpool status -x                       # want "all pools are healthy"
zfs list
zfs get -r mountpoint MediaTank AIVault DataPool
# normalise off /mnt if needed:
zfs set mountpoint=/MediaTank MediaTank   # (and AIVault, DataPool)
```

# Reboot survival (force-imported pools sometimes don't auto-import)

```bash
systemctl enable zfs-import-cache zfs-import.target
zpool set cachefile=/etc/zfs/zpool.cache MediaTank
zpool set cachefile=/etc/zfs/zpool.cache AIVault
zpool set cachefile=/etc/zfs/zpool.cache DataPool
```
Reboot once and confirm all three in `zpool list` BEFORE stacking containers on them.

# Bind mounts into containers

In the container conf (e.g. `/etc/pve/lxc/102.conf`):
```
mp0: /AIVault/ollama-models,mp=/mnt/models
```

# The UID-mapping permission dance (unprivileged LXC)

This is THE recurring storage gotcha. Unprivileged container root maps to host UID
**100000**. So to make a bind mount writable by the container's root-owned service, chown
**on the host** using the mapped ID — never `chown 1000` and never from inside the
container:
```bash
# on the HOST:
chown -R 100000:100000 /AIVault/ollama-models
```
If a service inside the container writes as a non-root user, add that user's in-container
UID to 100000 to get the host-side owner. For GROUP access to a device/mount (the
render-group case), map the group through with an idmap instead — see
[GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md).

# apt in any LXC

IPv6 mirrors fail → `apt -o Acquire::ForceIPv4=true update`, or persist:
```bash
echo 'Acquire::ForceIPv4 "true";' > /etc/apt/apt.conf.d/99force-ipv4
```

# Citations

[1] n5-pro-homelab Claude Skill — references/proxmox-storage.md (Dave's claude.ai account, last updated 2026-07-19)
