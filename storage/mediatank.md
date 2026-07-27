---
type: ZFS Pool
title: MediaTank
description: 5.5TB Seagate HDD for media and backups, bind-mounted only, not a Proxmox storage.
resource: /MediaTank
tags: [zfs, storage, media]
timestamp: 2026-07-25T00:00:00Z
---

5.45TB usable, Seagate HDD. Mountpoint `/MediaTank` (normalised off TrueNAS's
`/mnt/MediaTank`). 1.41T used / 4.04T free as of 2026-07-25.

Holds media and backups. **Not registered as Proxmox storage** — it is bind-mounted into
containers only, never carved into VM/LXC disk images. Used by
[nas (CT100)](/containers/nas.md), [jellyfin (CT101)](/containers/jellyfin.md),
[sabnzbd (CT108)](/containers/sabnzbd.md), and
[audiobooks (CT111)](/containers/audiobooks.md).

# `media/Audiobooks` — added 2026-07-26

Output directory for [audiobooks (CT111)](/containers/audiobooks.md), one subfolder per
book holding chaptered MP3s. Owned `100000:100112` mode `2775` — CT111's root owns it so
the converter can write, group `100112` matches the rest of the library so other media
containers can read, and the **setgid bit is load-bearing**: without it generated files
land in the wrong group. Source ebooks are read from the existing `media/Books`
(`100103:100112`), with `media/Books/inbox` added the same day as an SMB-writable staging
area for books awaiting conversion. See
[EPUB/PDF to audiobook](/playbooks/epub-to-audiobook.md).

# `Media` vs `media` — two different directories, case is the only difference (found 2026-07-25)

Both exist at the pool root and are genuinely different, not a mistake in one place:

* **`Media`** (132G) — owned by uid/gid `3000`. Contains a `Movies/` folder with content
  dated 2026-07-04 and untouched since, named like typical downloaded releases (e.g.
  `[WEBRip] [720p] [YTS.LT]`) — looks like an old download landing spot, not a Mac backup
  as first assumed. The `.DS_Store` present is just Finder having browsed it at some
  point, not proof of Mac origin.
* **`media`** (1.28T, the real library, actively growing — last touched 2026-07-20) —
  owned by uid/gid `100103:100112` (an unprivileged-LXC mapped UID, matching the
  [ZFS bind-mount permissions pattern](/playbooks/zfs-bind-mount-permissions.md)),
  contains `Audio/`, `Books/`, `Comics/`. This is what the containers actually read from.

**Checked 2026-07-25: not a live collision.** [nas (CT100)](/containers/nas.md)'s
`smb.conf` has exactly one share, `[media]` → `/srv/media` (the lowercase, active one) —
**`Media` isn't exposed over Samba at all right now**, so there's no SMB client picking
between them today. It reads as 132G of real, recoverable movie content that got orphaned
when `media` became the active library, not an active risk. Dave chose to leave it as-is
for now rather than touch it — see [actions.md](/actions.md).

# Leftover TrueNAS system datasets — removed 2026-07-25

`MediaTank/.system/{configs,cores,netdata,nfs,samba4,vm}-*` — tiny (under a few MB each),
auto-created by TrueNAS's own middleware before the pool was imported into Proxmox (see
[the host page](/host/n5-pro.md) — "virtualising TrueNAS was pointless and was removed").
Confirmed inert (all `mountpoint: legacy`, never auto-mounted, nothing reading them) and
removed: `zfs destroy -r MediaTank/.system`. Pool root now only holds `Media`, `media`,
and `backups`.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/proxmox-storage.md (Dave's claude.ai account, last updated 2026-07-19)
