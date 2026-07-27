---
type: LXC Container
title: audiobooks (CT111)
description: EPUB/PDF to audiobook converter, unprivileged LXC at 192.168.0.24, writing MP3s onto MediaTank.
tags: [proxmox, lxc, audiobooks, tts, kokoro, piper, calibre, media]
timestamp: 2026-07-26T00:00:00Z
---

CT111, hostname `audiobooks`, unprivileged LXC at **192.168.0.24**. Created 2026-07-26 to
turn EPUBs and PDFs into chaptered MP3 audiobooks using the
[Kokoro TTS service](/containers/docker-stack.md#kokoro-tts-added-2026-07-26) on
docker-stack. Usable either from the CLI or through a web UI at
**`https://audiobooks.lan`**.

For how to actually run a conversion, see
[EPUB/PDF to audiobook](/playbooks/epub-to-audiobook.md).

# Spec

Debian 13 (trixie) from `local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst`, 4 cores,
4GB RAM, 512MB swap, 16G rootfs on [DataPool](/storage/datapool.md) (2.0G used after
provisioning). `onboot: 1`. Static `.24` with DNS pointed at
[AdGuard](/containers/adguard.md) and IPv6 disabled via
`/etc/sysctl.d/99-disable-ipv6.conf`, matching
[fleet policy](/network/ip-addressing.md) at creation time rather than drifting into it
later.

**`features: nesting=1` is required, not optional.** `pct create` emits
`WARN: Systemd 257 detected. You may need to enable nesting.` — Debian 13's systemd needs
it in an unprivileged container. Set with `pct set 111 --features nesting=1`.

# Storage

`mp0: /MediaTank/media,mp=/srv/media` — the whole [MediaTank](/storage/mediatank.md)
media library bind-mounted, so source books and finished audiobooks both live on the pool
rather than in the container. This is the reason the converter is an LXC and not another
Docker service on VM103: a VM can't bind-mount a ZFS dataset, it would have needed a
CIFS/NFS round-trip back through [nas (CT100)](/containers/nas.md).

* **Input**: `/srv/media/Books` (host `/MediaTank/media/Books`), read-only in practice.
  `Books/inbox/` is the staging area for books awaiting conversion, writable over SMB via
  the existing `[media]` share on [nas (CT100)](/containers/nas.md).
* **Output**: `/srv/media/Audiobooks` (host `/MediaTank/media/Audiobooks`), created
  2026-07-26.

# The output directory's ownership is deliberate

Set up on the **host**, using mapped UIDs, per the
[bind-mount permissions playbook](/playbooks/zfs-bind-mount-permissions.md):

```bash
# on the HOST:
mkdir -p /MediaTank/media/Audiobooks
chown 100000:100112 /MediaTank/media/Audiobooks
chmod 2775 /MediaTank/media/Audiobooks
```

* Owner `100000` = **CT111's root**, so the converter can write without any idmap work.
* Group `100112` = the same group the rest of the library already uses (`Books`,
  `Comics`, `Movies` are all `100103:100112`), so other media containers can read the
  results.
* The **setgid bit (`2775`) is the part that matters** — without it, generated MP3s land
  in CT111's root group and other containers lose group access. Verified working: the
  first generated file came out `100000:100112`, group inherited.

# Installed software

* **[epub_to_audiobook](https://github.com/p0n1/epub_to_audiobook)** at
  `/opt/epub_to_audiobook`, in its own venv (`venv/bin/python`) — Debian 13 is
  PEP-668 externally-managed, so a venv is required, not a preference.
* **calibre 8.5.0** for `ebook-convert`, the PDF→EPUB step. epub_to_audiobook itself
  takes **EPUB only** — PDF support is entirely calibre's.
* **ffmpeg 7.1.5**, used for MP3 muxing.
* **Piper** (`piper-tts 1.6.0`, the OHF-Voice `piper1-gpl` line, with `onnxruntime 1.28.0`)
  in a **second, separate venv** at `/opt/piper1/venv` — added 2026-07-26. Deliberately not
  installed into epub_to_audiobook's venv: they pull different pins, and Piper is the
  backend most likely to be upgraded independently. Voices live under
  `/opt/piper/espeak-ng-data/voices` (~61MB each), which is not a path anyone would choose
  — the provider derives it from the executable's parent directory. `/opt/piper` is 348MB
  with five voices installed, `/opt/piper1` another 118MB, against a 16G rootfs at 16% used.
* Python 3.13.5.

# Piper: three files, and only one of them is obvious

* **`/opt/piper1/venv/bin/piper`** — the actual binary. Nothing should point at this
  directly.
* **`/opt/piper/piper`** — a shell **adapter** that everything points at instead. It works
  around two upstream argv bugs and pins the thread count; it's also what fixes the voice
  directory in place, since the provider computes that path from wherever this file sits.
  Full reasoning in
  [the playbook](/playbooks/epub-to-audiobook.md#optpiperpiper-is-an-adapter-script-not-the-binary).
* **`/usr/local/bin/piper-voice`** — installs a voice correctly. epub_to_audiobook's own
  downloader writes the `.onnx.json` over the `.onnx` path and leaves the voice unusable,
  so this exists to not use it.

`/opt/piper/piper.bak-20260726` and `/usr/local/bin/audiobook-piper.bak-20260726` are
supersede backups from the same day's work — safe to delete.

# `/usr/local/bin/audiobook`

A wrapper that does PDF detection, calibre conversion, and the TTS call in one command.
It hardcodes the two flags that are easy to get wrong:

* `--model_name tts-1` — **required**. Kokoro breaks on epub_to_audiobook's default
  model name.
* `--no_prompt` — the tool otherwise blocks on an interactive `input()` confirmation and
  dies with `EOFError` the moment it's run non-interactively.

Voice, output root and Kokoro URL are environment overrides (`VOICE`, `OUTROOT`,
`KOKORO_URL`), defaulting to `af_bella`, `/srv/media/Audiobooks`, and
`http://192.168.0.14:8880/v1`.

**The wrapper is Kokoro-only by construction.** `--model_name tts-1` is meaningless to
every other backend the tool supports, so Edge and Azure conversions have to call
`venv/bin/python main.py` directly with `--tts <backend>`. This matters in practice
because Kokoro has no Irish voice and Edge does — see
[Irish voices](/playbooks/epub-to-audiobook.md#irish-voices-are-the-one-thing-kokoro-cant-do).

# `/usr/local/bin/audiobook-piper`

Sibling of `audiobook`, added 2026-07-26, same interface — PDF detection, calibre
conversion, TTS call — but pointed at local Piper instead of Kokoro. `VOICE` defaults to
`en_GB-alba-medium`; `OUTROOT` and `PIPER` are the other overrides.

It installs a missing voice for you by calling `piper-voice` up front rather than letting
the provider's broken downloader run. That call **must be an absolute path**: `pct exec`
runs with `PATH=/sbin:/bin:/usr/sbin:/usr/bin`, which does not include `/usr/local/bin`, so
the original bare `piper-voice` would have failed with "command not found" the first time
anyone asked for an uninstalled voice. Fixed and verified the same day.

# Web UI — `audiobook-ui.service`

epub_to_audiobook ships a **Gradio UI** (`main_ui.py`) that the pip install already pulls
in via its `gradio` dependency. Run as `audiobook-ui.service` (enabled at boot), listening
on `0.0.0.0:7860`, reverse-proxied to `https://audiobooks.lan` by
[Caddy](/playbooks/reverse-proxy-caddy.md).

Three deliberate choices in the unit file, each of which is a bug if you get it wrong:

* **`WorkingDirectory=/srv/media/Audiobooks`.** The UI's default output path is
  *relative* (`audiobook_output/<timestamp>`), so with the obvious working directory of
  `/opt/epub_to_audiobook` every conversion would silently land on the container's 16G
  rootfs instead of the pool. Pointing CWD at the bind mount makes the default correct.
* **Both `OPENAI_BASE_URL` and `OPENAI_API_BASE` are set** to Kokoro. The `openai` SDK
  reads the former; this app's own UI text documents the latter. Setting only one risks
  the UI quietly falling back to **the real OpenAI API** — a paid endpoint — instead of
  erroring. Both are set so that can't happen.
* **`GRADIO_ANALYTICS_ENABLED=False`** and `HF_HUB_DISABLE_TELEMETRY=1`, since neither
  Gradio nor HuggingFace needs to phone out from this LAN.

Verify the env actually reached the process rather than trusting the unit file — this
bundle has been bitten by that before:

```bash
pct exec 111 -- systemctl show audiobook-ui -p Environment | tr ' ' '\n' | grep -i base
```

# Citations

[1] Direct build and live verification on this host, 2026-07-26 — container creation,
provisioning, and an end-to-end conversion test.

[2] Piper install and live verification on CT111, 2026-07-26 — package versions, on-disk
layout and sizes read off the running container, plus end-to-end conversions through
`audiobook-piper`.
