---
type: Playbook
title: EPUB/PDF to audiobook
description: Turning ebooks into chaptered MP3s with epub_to_audiobook on CT111, across three TTS backends — Kokoro, local Piper, and Edge.
tags: [audiobooks, tts, kokoro, piper, edge-tts, calibre, lxc, docker]
timestamp: 2026-07-26T00:00:00Z
---

Three backends are available; two of them are local. Two pieces, deliberately on different
guests:

* **Kokoro-FastAPI** (`stack-kokoro-1`) on
  [docker-stack (VM103)](/containers/docker-stack.md) — an OpenAI-compatible TTS server
  on `192.168.0.14:8880`. It's a network service that never touches the filesystem.
* **epub_to_audiobook** on [audiobooks (CT111)](/containers/audiobooks.md) — a batch CLI
  that needs direct [MediaTank](/storage/mediatank.md) access, which only an LXC can have.
* **Piper** (added 2026-07-26), installed *inside* CT111 rather than as a network service —
  no server, no container, just a binary and a voice file. See
  [Piper](#piper--the-fastest-local-option) below; it's the fastest of the three.

# Getting a book onto the box

Over SMB from the Mac — the existing `[media]` share on
[nas (CT100)](/containers/nas.md) already covers it, no new share needed:

```
smb://192.168.0.11/media        (Finder: Cmd-K, connect as "dave")
```

Drop the `.epub` or `.pdf` into **`Books/inbox/`** (created 2026-07-26 as a staging area so
books awaiting conversion don't get lost among the 45 existing library folders). The same
share is where finished audiobooks appear, under `Audiobooks/`.

This works because the Samba user `dave` (uid 1000 in CT100) is a member of group
**112 (`sharedmedia`)**, and the library tree is `2775` group-writable to that gid —
verified 2026-07-26 by actually creating a file as `dave`, not just by reading the mode
bits. Any new staging directory must keep group `100112` and the setgid bit on the host
or SMB writes will start failing.

The path CT111 sees is `/srv/media/Books/inbox` (host `/MediaTank/media/Books/inbox`).

# The web UI (easiest path)

**`https://audiobooks.lan`** — upload a book, pick a voice, hit Start. Served by
`audiobook-ui.service` on [CT111](/containers/audiobooks.md).

Four things trip you up on first use, all of them defaults that assume you're paying
OpenAI rather than running Kokoro locally:

1. **The TTS tab defaults to `Edge`, not `OpenAI`.** Click **OpenAI** or you'll get
   Microsoft Edge voices and never touch Kokoro at all.
2. **Change Model from `gpt-4o-mini-tts` to `tts-1`.** The default fails against Kokoro —
   same trap the CLI wrapper works around with `--model_name tts-1`. `tts-1-hd` also
   works; the "HD"/pricing distinctions in the dropdown are OpenAI's and mean nothing
   here.
3. **The Voice list shows OpenAI names** (`alloy`, `nova`, `shimmer`…), not Kokoro's
   `af_*`/`bf_*` names. That's fine — Kokoro maps them internally, verified 2026-07-26 by
   generating real audio with `voice=alloy`. To use a specific Kokoro voice by name, use
   the CLI instead.
4. **Upload accepts `.epub` only.** PDFs must go through the CLI wrapper, which runs
   calibre first.

**Tick "Enable Preview Mode" for a first pass** — it parses chapters and estimates cost
without generating audio.

Output defaults to `audiobook_output/<timestamp>/` **relative to
`/srv/media/Audiobooks`**, so it lands on the pool; type an absolute path in "Set Output
Directory" to control it. The UI writes its own logs to
`/srv/media/Audiobooks/logs/`.

A long conversion is tied to the browser session's progress view, but the work happens
server-side in the service — for a full-length book the CLI + `systemd-run` route below is
still the more robust option.

# Running a conversion from the CLI

From the host (`pct exec` doesn't resolve `PATH`, so use the full path):

```bash
pct exec 111 -- /usr/local/bin/audiobook "/srv/media/Books/Some Book.epub"
```

Output goes to `/srv/media/Audiobooks/<book name>/` as one MP3 per chapter. Optional
second argument overrides the output folder name; anything after that is passed straight
through to `epub_to_audiobook`:

```bash
# pick a voice, and only do chapters 5-10
pct exec 111 -- env VOICE=bf_emma /usr/local/bin/audiobook \
    "/srv/media/Books/Some Book.pdf" some-book --chapter_start 5 --chapter_end 10
```

**Always `--preview` first on an unfamiliar book.** It enumerates the chapters it found
without generating any audio, which is how you catch a book whose front matter is split
into 20 junk "chapters":

```bash
pct exec 111 -- /usr/local/bin/audiobook "/srv/media/Books/Some Book.epub" x --preview
```

## Long books: detach the job

A full book takes hours, so don't run it in a foreground `pct exec` that dies with your
shell. Backgrounding with `nohup` inside `pct exec` **does not work** — the process is
killed when the exec session tears down (learned the hard way; the job silently never
ran). Use `systemd-run` inside the container:

```bash
pct exec 111 -- systemd-run --unit=ab-somebook --collect /bin/bash -c \
  '/usr/local/bin/audiobook "/srv/media/Books/Some Book.epub" > /root/ab.log 2>&1'

pct exec 111 -- systemctl is-active ab-somebook   # active | inactive
pct exec 111 -- tail -5 /root/ab.log
```

# Speed

Kokoro runs on **CPU**, and that is the right call here — the Radeon 890M is already
shared by [ollama](/containers/ollama.md) and [jellyfin](/containers/jellyfin.md), Kokoro
ships no Vulkan build, and its ROCm image is experimental and x86-only against a `gfx1150`
whose ROCm support this homelab has already found flaky (see
[GPU passthrough](/playbooks/gpu-passthrough-ollama-vulkan.md)). Kokoro-82M is small
enough that none of that matters.

All three backends measured on this box, 2026-07-26:

| Backend | Throughput | Notes |
|---|---|---|
| **Piper** | **~20× realtime** | Single stream, CPU, in-container. 3,540 chars → 227s of audio in 11.2s. |
| **Kokoro** | ~4.5–6× realtime | Single stream, CPU, over HTTP to VM103. A 14.7-min chapter took ~2.5 min. |
| **Edge** | network-bound | Parallelises instead — 8 workers, chapters landing every few seconds. |

So a 10-hour book is roughly **30 minutes on Piper** against **2 hours on Kokoro**. Piper is
a much smaller model doing much less work; the trade is voice quality, which is why Kokoro
is still the default. Audition before committing a long book to it.

# Piper — the fastest local option

Added 2026-07-26 as a second **fully local** backend. Unlike Kokoro it isn't a service at
all — no container, no port, no HTTP. `piper-tts 1.6.0` (the OHF-Voice `piper1-gpl` line)
lives in its own venv at `/opt/piper1/venv` inside CT111, and a voice is a single `.onnx`
file plus its `.onnx.json` config.

```bash
pct exec 111 -- env VOICE=en_GB-alba-medium /usr/local/bin/audiobook-piper \
    "/srv/media/Books/Some Book.epub"
```

Same wrapper conventions as `audiobook` — optional second argument names the output
folder, anything after that passes through, output lands in `/srv/media/Audiobooks/<name>/`,
PDFs go through calibre first. Long books still want the `systemd-run` treatment
[described above](#long-books-detach-the-job).

## Installing a voice

**Always use `piper-voice`, never the tool's own downloader.**

```bash
pct exec 111 -- /usr/local/bin/piper-voice en_GB-cori-medium
```

epub_to_audiobook's built-in voice download is broken: it writes both the `.onnx` and the
`.onnx.json` to the **same `.onnx` filename**, so the second download is skipped as
"already exists" and the voice arrives with no config at all. Piper then dies on a missing
config. `piper-voice` fetches both files to their correct names from the same HuggingFace
release. The `audiobook-piper` wrapper calls it automatically for a voice you don't have —
verified 2026-07-26 by requesting an uninstalled voice and watching both files land.

Five voices are installed: `en_GB-alba-medium` (Scottish, the wrapper default),
`en_GB-cori-medium`, `en_GB-northern_english_male-medium`, `en_US-lessac-medium`,
`en_US-ryan-high`. Budget ~61MB per medium voice (116MB for `high`) on the container's 16G
rootfs — `/opt/piper` is 348MB with those five, `/opt/piper1` another 118MB.

Coverage is **38 locales, but only `en_GB` and `en_US` in English** — 9 British voices and
18 American. There is no Irish voice, confirmed again 2026-07-26 against the live voice
index, which is the whole reason the Irish route goes through Edge.

## `/opt/piper/piper` is an adapter script, not the binary

This looks wrong and is deliberate. The provider derives the voice directory from
`dirname(piper_path)/espeak-ng-data/voices`, so where you point it decides where voices
live — hence voices sitting under an `espeak-ng-data` path that has nothing to do with
espeak. The adapter sits at that path and fixes three things before `exec`ing the real
binary:

1. **Left-trims argv.** The provider emits `" --noise_w"` with a leading space (upstream
   typo). argparse doesn't recognise it, and silently drops it *and its value* — so the
   Width Noise Scale slider is a no-op without the trim.
2. **Drops any flag whose value is the literal string `"None"`.** `--piper_noise_scale`
   and `--piper_noise_w_scale` **are not CLI flags at all** — they exist only in the UI, so
   on the command line they're unset and the provider `str()`s them straight into argv.
   piper1 then fails to parse `"None"` as a float, exits 2, and leaves a zero-byte WAV that
   ffmpeg refuses to decode (`invalid start code in RIFF header`). Dropping the pair lets
   the voice's own `.onnx.json` defaults apply.
3. **Pins `OMP_NUM_THREADS=4`.** onnxruntime still logs a `pthread_setaffinity_np` error
   per worker in an unprivileged LXC — this makes it 3 lines instead of one per core. It's
   noise, not a failure.

## The UI's Piper tab needs three things set by hand

Verified 2026-07-26 by reading the running UI's own code and the provider it calls:

1. **"Select Piper Deployment" has no default, and the Docker fields are the visible
   ones.** Leave it alone and the conversion tries to start a `linuxserver/piper` Docker
   container — **CT111 has no Docker at all**, so it fails outright. Pick **Local**.
2. **"Piper executable path" is an empty textbox** with no default. Type
   `/opt/piper/piper` — the adapter, not `/opt/piper1/venv/bin/piper`.
3. **The Language/Voice/Quality dropdowns will happily select a voice you haven't
   installed**, which drops you into the broken downloader above. Run `piper-voice` for it
   first.

The four sliders (Audio Noise Scale `0.667`, Width Noise Scale `0.8`, Audio Length Scale
`1.0`, Sentence Silence `0.2`) do have sane defaults, and the UI sends real numbers rather
than the CLI's `None` — that path was verified 2026-07-26 by driving the provider with the
slider defaults and getting real audio out.

# Voices (Kokoro)

Piper's are separate and named differently — see
[Installing a voice](#installing-a-voice) above.

`af_*` are American female, `bf_*` British female, `am_*`/`bm_*` the male equivalents.
Voices can be blended by name (`af_bella+af_sky`). Full list:

```bash
curl -s http://192.168.0.14:8880/v1/audio/voices
```

Kokoro also has a web UI at `https://kokoro.docker.lan/web/` and API docs at `/docs` for
auditioning voices before committing to a 10-hour render. **The trailing slash matters** —
`/web` 307-redirects to `/web/`, which some clients handle badly.

## Irish voices are the one thing Kokoro can't do

Checked live 2026-07-26, both local backends: **Kokoro has no Irish voice** (60-odd voices,
all `af_/am_/bf_/bm_/ef/em/ff/hf/hm/if/im/jf/…`, nothing `ie`), and **Piper has none
either** — 38 locales, English limited to `en_GB` and `en_US`, **no `en_IE`** and no `ga`.
Re-confirmed against the live voice index after Piper was actually installed, not just from
the dropdown. If you want an Irish narrator, the local path doesn't have one.

`en_GB-alba-medium` is Scottish, which is the closest local approximation and not close.

The `en-IE` that shows up in the UI is in the **Edge** tab (and Azure), and there is
**nothing to install** — these are Microsoft's free, unauthenticated cloud voices. No
model file, no download, no API key. Four exist:

| Voice | |
|---|---|
| `en-IE-ConnorNeural` | Hiberno-English, male |
| `en-IE-EmilyNeural` | Hiberno-English, female |
| `ga-IE-ColmNeural` | actual Irish (Gaeilge), male |
| `ga-IE-OrlaNeural` | actual Irish (Gaeilge), female |

In the UI: TTS tab → **Edge** → Language `en-IE` → the Voice dropdown repopulates with
Connor and Emily → Start. Verified 2026-07-26 by driving the running service's own
`get_edge_voices_by_language` handler, not by reading the dropdown.

**Understand what you're trading before you commit a book to this.** Edge sends the full
text of the book to Microsoft, chunk by chunk, over the internet — which is the exact
thing CT111 + Kokoro were built to avoid. Use it when you specifically want the accent;
stay on Kokoro otherwise. Speed is *not* the argument against it — see below.

**The `audiobook` wrapper cannot drive Edge.** It hardcodes `--model_name tts-1` against
Kokoro's OpenAI-compatible endpoint (see [Gotchas](#gotchas)), so Edge needs `main.py`
directly:

```bash
pct exec 111 -- systemd-run --unit=ab-irish --collect /bin/bash -c \
  'cd /opt/epub_to_audiobook && venv/bin/python main.py \
     --tts edge --language en-IE --voice_name en-IE-ConnorNeural --no_prompt \
     "/srv/media/Books/Some Book.epub" "/srv/media/Audiobooks/some-book" \
     > /root/ab-irish.log 2>&1'
```

`--no_prompt` is still required — that trap is the tool's, not Kokoro's. Note the output
directory is a positional argument here, not the `OUTROOT` env var the wrapper uses, so
give it a full path under `/srv/media/Audiobooks` or the MP3s land on the container's 16G
rootfs.

## Edge is verified working on this box

Confirmed 2026-07-26 by actually rendering audio, not by reading a dropdown — the voice
list in the UI is a static constant in the app (it answers in 0.7ms, no network call), so
it only ever proved the option was *offered*.

A one-chapter EPUB through **Edge / `en-IE-ConnorNeural`** produced:

```
Processing chapter-1_Smoke_Test_chunk_1_of_1, length=183
✅ Converted chapter 1: Smoke_Test,
   output file: /srv/media/Audiobooks/edge-ie-smoketest/0001_Smoke_Test.mp3
All chapters converted successfully.
```

183 characters in ~1.4s. So `edge_tts` is present in the venv and **CT111 has working
egress to Microsoft** — the two things that could have blocked this.

**Edge is fast, and faster than Kokoro here.** The same log shows a full 30-chapter book
(`the_reverse_cenators_guide_to_surivival_after_ai`) converted end-to-end through
`edge_tts_provider` on 2026-07-26, running **8 chapters concurrently** across separate
workers with chapters completing every few seconds. Kokoro's measured ~4.5–6× realtime is
a single-stream CPU figure; Edge parallelises across the network instead. Set **Worker
Count** in the UI accordingly — it defaults to 1, which throws away the entire advantage.

That run also used the UI's *relative* default output path and landed correctly under
`/srv/media/Audiobooks`, which is the `WorkingDirectory` choice in
[`audiobook-ui.service`](/containers/audiobooks.md#web-ui--audiobook-uiservice) working as
intended on a real book rather than in theory.

A leftover `Audiobooks/edge-ie-smoketest/` directory is the smoke test itself — delete it
whenever.

# Gotchas

* **`--model_name tts-1` is required.** epub_to_audiobook's default model name makes
  Kokoro fail. The wrapper sets it; if you call `main.py` directly, you must too.
* **`--no_prompt` is required for anything non-interactive.** Without it the tool calls
  `input()` for confirmation and dies with `EOFError: EOF when reading a line`. The
  wrapper sets it.
* **`OPENAI_API_KEY` must be set to something**, anything — the OpenAI client library
  refuses to start without it even though Kokoro ignores it. The wrapper defaults it to
  `not-needed`.
* **PDF conversion is lossy.** `ebook-convert` handles reflowable text fine, but
  multi-column layouts, tables and headers/footers come through as garbled narration, and
  a scanned PDF produces nothing at all without OCR. Check with `--preview` before
  committing. EPUB in, always, when you have the choice.
* **`pct exec`'s PATH is `/sbin:/bin:/usr/sbin:/usr/bin` — `/usr/local/bin` is not on
  it.** That's why every command here is given as a full path, and it bites *inside*
  scripts too, not just at the prompt: `audiobook-piper` originally called `piper-voice`
  bare and would have died with "command not found" the first time someone asked for a
  voice that wasn't already installed. Fixed 2026-07-26 to use the absolute path. Any new
  helper that shells out to another helper needs the same treatment.
* **The output dir's setgid bit matters** — see
  [audiobooks (CT111)](/containers/audiobooks.md#the-output-directorys-ownership-is-deliberate).
  Drop it and new MP3s stop being group-readable by the other media containers.

# Citations

[1] Direct build and live verification on this host, 2026-07-26 — including an
end-to-end conversion of one chapter and the timing figures above.

[2] Piper install and verification on CT111, 2026-07-26 — end-to-end conversions through
the wrapper (including the auto-download path with a deliberately-uninstalled voice), the
UI's slider values driven straight at the provider, and the ~20× realtime measurement.
