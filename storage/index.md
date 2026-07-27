# Storage

Three ZFS pools imported from a retired TrueNAS install (plain ZFS — Proxmox reads them
natively; virtualising TrueNAS was pointless and was removed). Mountpoints were
normalised off TrueNAS's `/mnt/` prefix to root paths.

* [MediaTank](mediatank.md) - 5.5TB Seagate HDD, media + backups, bind-mounted only.
* [AIVault](aivault.md) - 512GB WD NVMe, mostly Ollama models (Postgres/Qdrant provisioned here but not actually in use).
* [DataPool](datapool.md) - 1TB Samsung SSD, VM/LXC rootfs, Docker volumes, and (in practice) docker-stack's real Postgres/Qdrant data.

For the reboot-survival and UID-mapping procedures shared across all three pools, see
[ZFS pools & bind-mount permissions](/playbooks/zfs-bind-mount-permissions.md).

# Health (checked 2026-07-25)

All three pools: **single disk, no redundancy** (no mirror/raidz — a disk failure loses
that pool entirely, not just degrades it). All `ONLINE`, no errors, scrubs run
automatically and clean (`scrub repaired 0B ... 0 errors`, last Jul 12). All three also
report "some supported and requested features are not enabled" (`zpool upgrade` available
but not run) — likely just never done since the TrueNAS import, not investigated further.
