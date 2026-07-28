---
type: LXC Container
title: audiobookshelf (CT112)
description: Audiobookshelf audiobook server, unprivileged LXC at 192.168.0.26, serving the MediaTank Audiobooks library at https://audiobookshelf.lan.
resource: /audiobooks
tags: [proxmox, lxc, audiobookshelf, audiobooks, media, server]
timestamp: 2026-07-26T00:00:00Z
---

CT112, hostname `audiobookshelf`, unprivileged LXC at **192.168.0.26**. Created 2026-07-26
to *serve and play* the audiobook library over the web and the Audiobookshelf mobile apps,
at **`https://audiobookshelf.lan`** — or **`https://audiobookshelf.133gsl.ie`**, which
serves a publicly-trusted Let's Encrypt certificate and so needs no per-device CA trust.
Preferred, especially on phones and the mobile apps. See
[133gsl.ie on Cloudflare DNS](/playbooks/dns-cloudflare-133gsl-ie.md).

**It is not the converter.** [audiobooks (CT111)](audiobooks.md) *produces* MP3s into
`/MediaTank/media/Audiobooks`; CT112 *reads that same directory* and serves it. The two
compose over one shared folder and know nothing about each other — the similar names are
the only thing they have in common, and `audiobooks.lan` vs `audiobookshelf.lan` are
different services on different guests.

# Spec

Debian 13 (trixie) from `local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst`, 2 cores,
2GB RAM, 512MB swap, 16G rootfs on [DataPool](/storage/datapool.md) (549M used after
install). `onboot: 1`, `features: nesting=1`, unprivileged. Static `.26`, DNS at
[AdGuard](adguard.md) (`192.168.0.20`), IPv6 disabled via
`/etc/sysctl.d/99-disable-ipv6.conf` at provisioning time — built to
[fleet policy](/network/ip-addressing.md) from the start, following CT111's precedent
rather than drifting into compliance later.

Verified to survive `pct reboot`: service back `active`, bind-mount ownership unchanged,
IPv6 still zero addresses.

# Why `.26` and not `.25`

`.25` was the obvious next static after CT111's `.24` — **and it is already occupied**.
A ping sweep before allocating found `192.168.0.25` answering, MAC `5c:1b:f4:88:46:08`,
a device that appears nowhere in this bundle's ledger. That is exactly the
[open DHCP-conflict problem](/network/ip-addressing.md#the-dhcp-conflict-problem-open)
landing on a live build, and it is the reason the "ping first" step in that page is not
ceremonial. `.26` was confirmed silent and taken instead. `.15` is also free — it is the
one gap inside the contiguous `.11`–`.24` static block — but it sits in the historical
`.12`/`.14`/`.16` collision zone that CT111 deliberately built above, so it was left alone.

The unidentified `.25` occupant is filed in [actions.md](/actions.md).

# Storage — deliberately narrower than CT111

`mp0: /MediaTank/media/Audiobooks,mp=/audiobooks` — **only the Audiobooks directory**, not
the whole media tree.

CT111 mounts all of `/MediaTank/media` because a converter legitimately needs `Books/` as
input. An audiobook server has no business seeing `Books`, `Comics`, `Movies`, `Porn` or
`Nudes`, so it does not get them. If podcasts are added later, mount
`/MediaTank/media/Audio` as a second `mp1` rather than widening `mp0` back out to the
parent.

No `chown` was needed on the host. The directory was already
`100000:100112 / drwxrwsr-x` from CT111's setup, and CT112 uses the **default idmap**
(no `idmap:` line in `pct config`), so both containers' root maps to host `100000` and the
directory presents inside CT112 as `0:112`.

# The service user, and why it is in group 112

The deb creates a system user `audiobookshelf` (**uid 999, gid 991**, shell `/bin/false`) —
it does **not** run as root, so container-root owning the mount buys it nothing. Against
mode `2775` it initially had only the *other* bits, `r-x`.

That is already enough to scan and stream, and was verified so before anything was
changed. It was still put in the mapped media group, because a read-only-by-accident
library fails confusingly the first time someone enables Audiobookshelf's
"store metadata/cover with item" settings, which write into the library folder:

```bash
pct exec 112 -- groupadd -g 112 medialib
pct exec 112 -- usermod -aG medialib audiobookshelf
```

`medialib` is a **local name for the mapped GID**, invented here — GID 112 in-container is
host GID `100112`, the group `Books`/`Comics`/`Movies` already share. Nothing outside CT112
knows that name; the number is what matters.

# Install — the official one-liner was not used

Audiobookshelf **2.35.1**, from the project's apt repo at
`https://advplyr.github.io/audiobookshelf-ppa`.

Two deviations from the upstream instructions, both deliberate:

* **`https://audiobookshelf.org/install.sh` does not exist.** It returns the site's HTML
  landing page with **HTTP 200** (SPA catch-all), so `curl -fsSL … | bash` is not caught by
  `curl -f` — it pipes HTML into a root shell. There is no official piped installer; the
  supported path is the apt repo.
* **The key is scoped to this repo.** Upstream says to drop it in
  `/etc/apt/trusted.gpg.d/`, which trusts it for *every* repository on the box. It went to
  `/usr/share/keyrings/audiobookshelf-archive-keyring.gpg` with a matching `signed-by=` in
  the sources line instead:

```
deb [signed-by=/usr/share/keyrings/audiobookshelf-archive-keyring.gpg] https://advplyr.github.io/audiobookshelf-ppa ./
```

  Upstream's own `audiobookshelf.list` omits `signed-by`, so this file is hand-written and
  will be **overwritten if the package ever ships its own** — re-check after a major
  upgrade.

# Config and metadata live on the rootfs, on purpose

`/etc/default/audiobookshelf` holds the defaults, unchanged:

```
METADATA_PATH=/usr/share/audiobookshelf/metadata
CONFIG_PATH=/usr/share/audiobookshelf/config
PORT=13378
HOST=0.0.0.0
```

CT111 had to move its output off the rootfs because audiobook MP3s are gigabytes per book.
Audiobookshelf's metadata is the SQLite DB plus cover art — megabytes, not gigabytes — so
the 16G rootfs is genuinely adequate and a second bind mount would be complexity for
nothing. This is a judgement about *sizes*, not a different rule: if the library grows to
where `du -sh /usr/share/audiobookshelf` is a meaningful fraction of 16G, move it then.

# Reverse proxy

`audiobookshelf.lan` → `192.168.0.26:13378`, via
[Caddy on docker-stack](/playbooks/reverse-proxy-caddy.md). Plain `reverse_proxy` with no
`transport` block — Audiobookshelf's websockets work through Caddy 2 untouched, same as the
Gradio UI on CT111 and unlike `proxmox.lan`. AdGuard rewrite answers `192.168.0.14`
(docker-stack), not `.26` — see the note in
[DNS via AdGuard](/network/dns-adguard.md) about why a block of identical `.14` answers is
correct by design here.

# Not yet done — first-run setup

The server reports `"isInit": false`. **Creating the admin account and adding the library
is deliberately left to Dave** — it sets a credential, which is not something to do on
someone's behalf.

At `https://audiobookshelf.lan`: create the admin user, then add a library pointing at
**`/audiobooks`** (the in-container path, not the host path).

One thing to expect on first scan: `/MediaTank/media/Audiobooks` currently also contains
CT111's leftover test artifacts — `rory-test/`, `edge-ie-smoketest/`, `piper-smoketest/`,
`audiobook_output/` and a `logs/` directory. Audiobookshelf will happily present those as
library items. They are all listed as safe to delete in [actions.md](/actions.md); clearing
them before the first scan avoids a library full of junk entries.

# Citations

[1] Direct build and live verification on this host, 2026-07-26 — container creation,
install, permissions, Caddy/AdGuard wiring, and an end-to-end `https://audiobookshelf.lan`
check plus a reboot-survival test.
[2] Audiobookshelf Debian install documentation —
https://www.audiobookshelf.org/docs/documentation/install/linux/linux-install-deb
(retrieved 2026-07-26), and the live repo metadata at
`https://advplyr.github.io/audiobookshelf-ppa`.
