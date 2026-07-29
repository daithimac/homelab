---
type: ZFS Pool
title: AIVault
description: 512GB WD NVMe holding the Ollama model store and (as of 2026-07-25) docker-stack's Postgres/Qdrant data.
resource: /AIVault
tags: [zfs, storage, ai, ollama]
timestamp: 2026-07-29T00:00:00Z
---

512GB WD NVMe (472G usable). Mountpoint `/AIVault`. Registered as Proxmox ZFS storage (disk
images + container rootfs). **89.8G used / 368G free as of 2026-07-29**, after the
`refreservation` on `vm-103-disk-0` was dropped — see below. Between 2026-07-26 and that
change it read 294G used / 164G free, of which 203G was reservation rather than data.

The Ollama model store at `AIVault/ollama-models` (89.7G, the pool's only real consumer),
bind-mounted into [ollama (CT102)](/containers/ollama.md) at `/mnt/models`
(`OLLAMA_MODELS=/mnt/models`), is confirmed live. Its contents were inventoried for the
first time on 2026-07-29 — seven models, all genuinely in use, no orphaned bulk; the
breakdown is on [the ollama page](/containers/ollama.md).

# Postgres/Qdrant migrated here 2026-07-25

Found unmounted and idle during a fleet-wide storage audit (`AIVault:vm-103-disk-0`, 200G,
attached to [docker-stack (VM103)](/containers/docker-stack.md) as `virtio1` but never
actually wired in — bare `LVM2_member`, no fstab entry). The real Postgres/Qdrant data was
sitting on the VM's root disk (`DataPool:vm-103-disk-1`) instead, as plain directories —
tiny at the time (67M total) but on the wrong pool for future growth. Dave chose to wire
the disk in rather than reclaim it.

**Migration, done live with the containers briefly stopped for consistency:**
`wipefs -a` the stale LVM signature, `mkfs.ext4` (label `aivault-data`), mounted at
`/mnt/aivault` (via `/etc/fstab`, UUID-pinned). `rsync -aHAX` copied `/data/postgres` and
`/data/qdrant` into `/mnt/aivault/{postgres,qdrant}/`, verified byte-identical with
`diff -rq` before touching anything. Originals renamed aside
(`/data/postgres.old-20260725`, `/data/qdrant.old-20260725` — kept as a safety net, not
yet deleted), fresh empty dirs created at the original paths, then bind-mounted onto the
new disk via two more `/etc/fstab` entries — meaning `/opt/stack/docker-compose.yml`
needed **zero changes**, since `/data/postgres` and `/data/qdrant` still resolve to real
directories from the container's point of view.

Verified end-to-end after restarting the containers: Postgres log shows it recognized the
existing database and skipped re-initialization, reached "ready to accept connections";
Qdrant came up cleanly on 6333/6334; n8n (which depends on Postgres) answered `200`
throughout. Two small orphaned datasets, `AIVault/postgres` and `AIVault/models` (96K
each, leftover from whatever the original unwired plan was), are now genuinely obsolete —
low-priority cleanup candidates whenever convenient. Re-verified empty 2026-07-29 and
deliberately left in place; see below.

# The 203G reservation — found 2026-07-26, released 2026-07-29

The ~200G jump between 2026-07-25 and 2026-07-26 was **not** data growth. `vm-103-disk-0`
— the 200G disk wired into docker-stack during the migration above — was **thickly
provisioned**, because the AIVault storage is declared `sparse 0` in
`/etc/pve/storage.cfg`. Proxmox therefore set a `refreservation` equal to the full volume
size the moment the disk was actually put to use, so 203G of the pool was spoken for to
store 19.8M, and **AIVault's "164G free" was the real ceiling for Ollama models**.

**Resolved 2026-07-29 by clearing the reservation on the existing zvol:**

```bash
zfs set refreservation=none AIVault/vm-103-disk-0
```

Before / after:

| | refreservation | used | referenced | volsize |
|---|---|---|---|---|
| before | 203G | 203G | 19.8M | 200G |
| after | none | 19.8M | 19.8M | 200G |

Pool free space went **164G → 368G** in one command, with no downtime and no filesystem
operation. This was chosen over the two options [actions.md](/actions.md) originally
listed because both were worse: shrinking the volume requires `resize2fs` on the ext4
filesystem *inside* a live VM disk **before** `zfs set volsize`, and getting the order
wrong truncates the filesystem; and setting `sparse 1` in `/etc/pve/storage.cfg` only
governs **future** disk creation, doing nothing about the 203G already reserved. Clearing
`refreservation` retroactively thin-provisions the existing zvol and is reversible in one
command (`zfs set refreservation=203G AIVault/vm-103-disk-0`).

Verified after the change with the same three checks used for the original migration:
Qdrant `200` on 6333 (`/collections` answering), n8n `200` on 5678, Postgres accepting TCP
on 5432, VM103 still `running`.

**The trade-off, stated plainly:** the reservation was what guaranteed docker-stack's disk
could not be starved. Without it, the VM's writes could fail if AIVault fills. Against
19.8M of actual use on a 200G volume — and the pool's only real consumer being a model
store that is now nowhere near the ceiling — that risk is remote, but it is no longer
*structurally* impossible. It is the reason to add a pool-usage alert on
[grafana (CT110)](/containers/grafana.md), which already scrapes the host via
`pve-exporter` and already has the "N5 Pro Host Overview" dashboard carrying a storage
pool usage panel; the alert rule itself is not yet built and is tracked in
[actions.md](/actions.md).

Two small orphaned datasets, `AIVault/postgres` and `AIVault/models` (96K each), were
re-verified as genuinely empty on 2026-07-29 — no contents beyond `.`/`..`, no snapshots,
referenced by no guest config in `/etc/pve/lxc/` or `/etc/pve/qemu-server/`. Dave chose to
leave them in place rather than destroy them; they remain inert and cost 192K total.

# Permissions

Bind-mount ownership for `ollama-models` must be set on the **host** using the
unprivileged-container mapped UID, never from inside the container — see
[ZFS pools & bind-mount permissions](/playbooks/zfs-bind-mount-permissions.md).

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/proxmox-storage.md, references/gpu-ollama.md (Dave's claude.ai account, last updated 2026-07-19)
[2] direct host review 2026-07-29 — `zfs set refreservation=none`, `zfs list -r AIVault`, `zfs get` before/after, docker-stack service checks (reservation release and verification)
