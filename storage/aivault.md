---
type: ZFS Pool
title: AIVault
description: 512GB WD NVMe holding the Ollama model store and (as of 2026-07-25) docker-stack's Postgres/Qdrant data.
resource: /AIVault
tags: [zfs, storage, ai, ollama]
timestamp: 2026-07-25T00:00:00Z
---

512GB WD NVMe (472G usable). Mountpoint `/AIVault`. Registered as Proxmox ZFS storage (disk
images + container rootfs). **294G used / 164G free as of 2026-07-26** — was 90.6G used /
381G free the previous day, see below for why that jumped ~200G overnight without anyone
storing 200G of anything.

The Ollama model store at `AIVault/ollama-models` (90.6G, the pool's only real consumer),
bind-mounted into [ollama (CT102)](/containers/ollama.md) at `/mnt/models`
(`OLLAMA_MODELS=/mnt/models`), is confirmed live.

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
low-priority cleanup candidates whenever convenient.

# The pool is 62% full but nearly all of it is a reservation (found 2026-07-26)

The ~200G jump between 2026-07-25 and 2026-07-26 is **not** data growth. `vm-103-disk-0`
— the 200G disk wired into docker-stack during the migration above — is **thickly
provisioned**, because the AIVault storage is declared `sparse 0` in
`/etc/pve/storage.cfg`. Proxmox therefore set a `refreservation` equal to the full volume
size the moment the disk was actually put to use:

```
zfs get refreservation,used,referenced,volsize AIVault/vm-103-disk-0
refreservation  203G
used            203G
referenced      19.7M     # <-- actual data
volsize         200G
```

So 203G of the pool is spoken for to store 19.7M. Nothing is broken and nothing is at
risk — a reservation is exactly what guarantees the VM can't be starved of its disk — but
it means **AIVault's "164G free" is the real ceiling for Ollama models**, and the pool
will read as alarmingly full forever without this context. Flagged in
[actions.md](/actions.md) as a decision (shrink the volume, or switch the storage to
sparse) rather than fixed unilaterally, since either change affects a live VM disk.

# Permissions

Bind-mount ownership for `ollama-models` must be set on the **host** using the
unprivileged-container mapped UID, never from inside the container — see
[ZFS pools & bind-mount permissions](/playbooks/zfs-bind-mount-permissions.md).

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/proxmox-storage.md, references/gpu-ollama.md (Dave's claude.ai account, last updated 2026-07-19)
