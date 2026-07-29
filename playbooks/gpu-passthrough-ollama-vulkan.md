---
type: Playbook
title: GPU passthrough & Ollama on Vulkan (CT102)
description: The full iGPU → Ollama-on-Vulkan chain for CT102 — device passthrough, idmap, Mesa backports, and the IGPU_ENABLE flag.
tags: [gpu, ollama, vulkan, lxc, proxmox]
timestamp: 2026-07-29T00:00:00Z
---

The Radeon 890M (RDNA 3.5, `gfx1150`, reported as `GFX1150`/`0x150e`) on
[the N5 Pro host](/host/n5-pro.md) drives [ollama (CT102)](/containers/ollama.md) via the
**Vulkan** backend (RADV), not ROCm — ROCm's gfx1150 support is flaky; Vulkan is the
working path here. This chain was fought end-to-end and every link matters. Symptoms that
send you here: `library=cpu`, `total_vram="0 B"`, `dropping integrated GPU`, or slow
inference.

# Steps

### 1. Device passthrough (host: `/etc/pve/lxc/102.conf`)
```
lxc.cgroup2.devices.allow: c 226:1 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```
iGPU is `card1` (`226:1`), render node `renderD128` (`226:128`).

### 2. Unprivileged idmap for the render group (host: `102.conf`)
Inside the container the render group is **GID 104**; on the host it's **GID 993**
(`getent group render` on host to confirm — don't assume). The `ollama` service user is
already a member of render inside the container, so once the group maps through, the
service gets device access with no further membership fiddling.

```
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 104
lxc.idmap: g 104 993 1
lxc.idmap: g 105 100105 65431
```
The three `g` lines MUST sum to 65536 with no gap/overlap or the container won't start:
104 + 1 + 65431 = 65536. If the in-container render GID is not 104, recompute the
boundaries.

### 3. Authorise the host GID in subgid/subuid (host)
Proxmox refuses an idmap reaching outside root's authorised range — checks BOTH files
even for a group-only remap:
```bash
grep 993 /etc/subgid || echo "root:993:1" >> /etc/subgid
grep 993 /etc/subuid || echo "root:993:1" >> /etc/subuid
```
Then `pct stop 102 && pct start 102`. Verify the flip:
```bash
pct enter 102
ls -ln /dev/dri/renderD128    # group should now read 104, not 65534
```

### 4. Mesa must be new enough (inside CT102)
Bookworm base ships **Mesa 22.3.6** — predates RDNA 3.5, so RADV enumerates nothing and
you get `llvmpipe` (software, `PHYSICAL_DEVICE_TYPE_CPU`) only. Pull Mesa 25.x from
backports:
```bash
echo "deb http://deb.debian.org/debian bookworm-backports main non-free-firmware" > /etc/apt/sources.list.d/backports.list
apt -o Acquire::ForceIPv4=true update
apt install -y -t bookworm-backports mesa-vulkan-drivers libgl1-mesa-dri
```
Verify as the `ollama` user (root is NOT in render, so root's `vulkaninfo` will show
permission denied — that's expected, not a failure):
```bash
apt install -y vulkan-tools
su -s /bin/bash ollama -c 'vulkaninfo --summary'
```
Win = a device `PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU`, `driverName = radv`,
`AMD Radeon Graphics (RADV GFX1150)`, vendorID `0x1002`.

### 5. The flag Ollama needs to NOT drop the iGPU
Even with Vulkan seeing the card, Ollama **deliberately drops integrated GPUs by
default**. The log line is literally `dropping integrated GPU; to enable, set
OLLAMA_IGPU_ENABLE=1`. This was the final boss of the whole saga. Set it (and the tuning
knobs) in the systemd override:

```bash
mkdir -p /etc/systemd/system/ollama.service.d
cat > /etc/systemd/system/ollama.service.d/override.conf <<'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_ORIGINS=*"
Environment="OLLAMA_MODELS=/mnt/models"
Environment="OLLAMA_CONTEXT_LENGTH=16384"
Environment="OLLAMA_IGPU_ENABLE=1"
Environment="OLLAMA_KEEP_ALIVE=-1"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_MAX_QUEUE=32"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
EOF
systemctl daemon-reload
systemctl restart ollama
```

**This block is the live contents as of 2026-07-29**, transcribed from the running
container rather than reconstructed. It previously listed only 8 variables and gave
`OLLAMA_CONTEXT_LENGTH=12400`; the deployed override has 11 and runs at `16384`. The three
that were missing from this doc entirely — `OLLAMA_NUM_PARALLEL`, `OLLAMA_MAX_QUEUE`,
`OLLAMA_MAX_LOADED_MODELS` — were already set on the host, so **do not assume anything
here runs at Ollama's default**; check with `systemctl show` before reasoning about them.

Concurrency bounds, and why these values:

* `OLLAMA_MAX_LOADED_MODELS=1` — was `2` until 2026-07-29. With `KEEP_ALIVE=-1` that let
  two models sit pinned simultaneously and indefinitely; observed live at 8.8 GB
  (Moonlit-Mirage-12B) + 3.2 GB (Qwen3.5-4B) = 12 GB held with both idle. At `1`, loading
  a second model evicts the first (verified). Single-stream throughput is unaffected —
  see the Baseline table.
* `OLLAMA_NUM_PARALLEL=1` — one request slot, so each of the two independent clients
  (openwebui, sillytavern, both hitting `192.168.0.13:11434` directly) queues behind the
  other rather than splitting KV cache. `OLLAMA_MAX_QUEUE=32` absorbs the wait. Raising
  this to 2 would allow genuine concurrency at the cost of doubled KV allocation on a
  bandwidth-bound iGPU; deliberately left at 1.

Verify the flag actually reached the process (the check that never lies —
`systemctl edit` in nano has silently failed via a stray lock file before, so confirm the
live value):
```bash
systemctl show ollama -p Environment | tr ' ' '\n' | grep -i igpu    # want OLLAMA_IGPU_ENABLE=1
journalctl -u ollama --no-pager --since "30 sec ago" | grep -iE 'vulkan|vram|dropping|inference compute'
```
Win = `dropping` line GONE, and
`inference compute ... library=Vulkan ... type=iGPU total="48.0 GiB"`. (It reports ~48GB
because the iGPU can address system RAM as GTT, so capacity is NOT capped at the dedicated
UMA frame buffer — currently 32GB, see below — but see the speed note below.)

# Baseline

Measured 2026-07-29 on the live host, three runs per model, identical prompt
(`"Write a 200-word summary of how DNS resolution works."`) via
`pct exec 102 -- ollama run --verbose <model> "<prompt>"`, reading `eval rate`. Ollama
0.31.1, Vulkan/iGPU, `OLLAMA_CONTEXT_LENGTH=16384`, `OLLAMA_KV_CACHE_TYPE=q8_0`,
`OLLAMA_FLASH_ATTENTION=1`, BIOS UMA **32GB**. Both models reported `100% GPU` in
`ollama ps`.

| Model | Quant | Size | Ctx | Run 1 | Run 2 | Run 3 | Spread |
|---|---|---|---|---|---|---|---|
| `Moonlit-Mirage-12B-i1-GGUF:latest` | i1 (~q4) | 7.5 GB | 16384 | 10.94 tok/s | 10.93 tok/s | 10.94 tok/s | 0.1% |
| `Qwen3.5-4B-...-Heretic-Literotica-i1-GGUF:latest` | i1 (~q4) | 2.7 GB | 16384 | 23.88 tok/s | 23.34 tok/s | 23.84 tok/s | 2.3% |

Cold load adds ~2.2 s for the 12B, ~3.3 s for the 4B; warm reload is ~0.15 s. Prompt eval
runs 250–470 tok/s warm.

**This supersedes the previous `~3.2 tok/s` figure, which was wrong by roughly 3.4×** — not
merely unqualified. That number carried no model, quant, context or date, and no run in
this baseline reproduces anything near it. Treat it as retired; it should not be cited
again. Anything that was reasoned from "iGPU inference here is ~3 tok/s" is worth
re-examining, because the real 12B figure is ~11 tok/s.

Note the 4B's `eval count` varies wildly between runs (1810 / 8094 / 2853 tokens) because
the model ignores the word limit and rambles — **`eval rate` is the stable metric, total
duration is not**. Compare rates, never wall-clock, when re-running this.

To reproduce, use the same prompt and model tags and report all three runs. If the spread
exceeds a few percent, something else is contending for the iGPU and the numbers are not
comparable.

# Tuning notes

* `OLLAMA_FLASH_ATTENTION=1` + `OLLAMA_KV_CACHE_TYPE=q8_0` only matter once the GPU
  backend is live (they did nothing on CPU). q8_0 halves KV cache, nearly lossless — worth
  it with the 16384 context.
* `OLLAMA_KEEP_ALIVE=-1` pins models in memory (shows as `Forever` in `ollama ps` and a
  giant duration in logs — normal). With q8 cache + 16384 ctx on a shared-DDR5 iGPU, watch
  for memory pressure — it's the first suspect if loads crawl or OOM. This is why
  `OLLAMA_MAX_LOADED_MODELS` matters: pinning is indefinite, so the model cap is the only
  thing bounding total held memory.
* Discovery may set a huge `default_num_ctx` (e.g. 262144) because it sees ~48GB. The
  explicit `OLLAMA_CONTEXT_LENGTH=16384` overrides per-model; keep it set.

# The real performance lever: BIOS UMA frame buffer

Generation runs at **~10.9 tok/s for a 12B q4 and ~23.8 tok/s for a 4B q4** (see
[Baseline](#baseline) for the full method). iGPU inference is **memory-bandwidth bound** on
DDR5, so the ceiling is inherent, not a misconfig. More CPU cores don't help (bandwidth,
not compute, is the limit). The one hardware lever is dedicated VRAM in BIOS:

1. Shut down guests cleanly, reboot host, enter BIOS (`Del` or `F2`).
2. Path is firmware-dependent: `Advanced → AMD CBS → NBIO Common Options → GFX
   Configuration` (or `UMA Frame Buffer Size`), set UMA mode to
   `UMA_SPECIFIED`/`Manual`.
3. **Currently 32GB** (`dmesg | grep -i 'VRAM:'` → `VRAM: 32768M`, confirmed live
   2026-07-25 — this doc previously said "currently 2GB," which was stale; someone
   applied a BIOS change at some point without updating this line). This is *above* the
   documented sweet spot: **24GB** covers a q4 30B model (~18GB) + KV headroom while
   leaving 72GB for host + guests out of 96GB physical; 16GB if only running ≤13B models;
   32GB was flagged as "defensible but marginal past 24" even before this was checked —
   right now it's costing every other guest 8GB of headroom versus the 24GB target.
   **This doc previously claimed there was "no measured inference benefit over 24GB". That
   claim was never true and has been removed: 24GB has never been run.** The only
   measurements that exist — the [Baseline](#baseline) above — were taken at 32GB, so they
   are the A-side of an A/B whose B-side is still outstanding. See
   [actions.md](/actions.md) for the decision, and the A/B procedure below.
4. **The outstanding A/B.** With a repeatable baseline now in hand, the 24GB question is
   finally answerable. Set UMA to 24GB per step 2, confirm `dmesg | grep -i 'VRAM:'` reads
   `24576M` and `free -h` shows ~70Gi rather than 62Gi, then re-run the exact Baseline
   prompt and models and compare `eval rate`. Decision rule, fixed in advance so the result
   is not rationalised afterwards: **if 24GB costs less than ~5% on tok/s (i.e. the 12B
   stays above ~10.4 tok/s), keep 24GB** and bank the 8GB for the rest of the fleet;
   if it costs materially more, 32GB is vindicated and the item closes as "measured,
   keeping 32GB". Expect a modest difference either way — the iGPU addresses system RAM as
   GTT and reports ~48GB regardless, so this is about allocation cleanliness for hot
   weights, not capacity.
5. Save (`F10`), reboot. Verify on host: `dmesg | grep -i 'VRAM:'` reads back your target
   (e.g. `24576M` for 24GB).

Dedicated UMA gives cleaner allocation for hot weights than GTT spill, but it's a
**modest** gain, not the CPU→GPU night-and-day. A 30B model will still be slow; match
model size to the hardware (7–14B q4 is the pleasant range).

# Citations

[1] n5-pro-homelab Claude Skill — references/gpu-ollama.md (Dave's claude.ai account, last updated 2026-07-19)
[2] direct host review 2026-07-29 — `pct exec 102 -- ollama run --verbose` benchmark runs, `systemctl show ollama -p Environment`, `ollama list`/`ollama ps`, `dmesg | grep VRAM:` (Baseline section, live override contents, concurrency bounds)
