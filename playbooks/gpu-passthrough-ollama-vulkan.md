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
`inference compute ... library=Vulkan ... type=iGPU total="63.2 GiB"` (measured 2026-07-29;
this doc previously said `48.0 GiB`). The figure is **UMA + GTT** — 32 GB frame buffer plus
31.2 GiB of borrowable system RAM — so capacity is NOT capped at the UMA carve-out. See
[UMA and GTT](#uma-and-gtt--the-memory-pool-and-how-its-split) for how that splits and why
it matters more than UMA alone.

Note this host reports the pool correctly. Containerised Ollama on Strix has a known bug
(#16462) where it reports **2.0 GiB** instead; if you ever see that, it's the bug, not a
passthrough regression.

# Baseline

Figures below are **three-run means for all seven models**, measured 2026-07-29 with the
harness (`/root/bench-ollama.sh`, label `baseline-uma32-gtt31-ctx16k-kvq8`), identical
prompt (`"Write a 200-word summary of how DNS resolution works."`) via
`pct exec 102 -- ollama run --verbose <model> "<prompt>"`, reading `eval rate`. Ollama
0.31.1, Vulkan/iGPU, `OLLAMA_CONTEXT_LENGTH=16384`, `OLLAMA_KV_CACHE_TYPE=q8_0`,
`OLLAMA_FLASH_ATTENTION=1`, BIOS UMA **32GB**. All seven models reported `100% GPU` in
`ollama ps`.

| Model | Type | Quant | Size | t/s | Runs |
|---|---|---|---|---|---|
| `Qwen3.5-4B-...-Heretic-Literotica-i1-GGUF` | dense | i1 (~q4) | 2.7 GB | **23.54** | 23.91 / 23.46 / 23.24 (2.9% spread) |
| `gemma-4-26B-A4B-it-ultra-uncensored-heretic-i1-GGUF` | **MoE ~4B active** | Q4_K_M | 16 GB | **24.82** | 24.77 / 24.69 / 24.99 (1.2% spread) |
| `Melody1437-26B-A4B-v2.0-GGUF` | **MoE ~4B active** | — | 17 GB | **19.71** | 19.71 / 19.75 / 19.68 (0.4% spread) |
| `Moonlit-Mirage-12B-i1-GGUF` | dense | i1 (~q4) | 7.5 GB | **10.88** | 10.92 / 10.92 / 10.79 (1.2% spread) |
| `Qwen3.6-27B-Fable-Fusion-711-...-GGUF` | dense | — | 18 GB | **4.28** | 4.28 / 4.28 / 4.28 (0.0% spread) |
| `Dolphin-Mistral-24B-Venice-Edition-GGUF` | dense | **Q8_0** | 25 GB | **3.26** | 3.26 / 3.26 / 3.26 (0.0% spread) |
| `MistralRP-Noromaid-NSFW-Mistral-7B-GGUF` | dense | **Q8_0** | 7.7 GB | **10.43** | 10.43 / 10.43 / 10.44 (0.1% spread) |

Cold load adds ~2.2 s for the 12B, ~3.3 s for the 4B, ~15 s for the 26B MoE and ~20 s for
the 24B; warm reload is ~0.15 s. Prompt eval runs 250–470 tok/s warm.

## The harness

Benchmarks are taken with `/root/bench-ollama.sh` on the host, which appends to
`/root/bench-results.csv`. Always pass a label describing the config under test:

    /root/bench-ollama.sh <label> [runs] [model-substring]

Compare rows by label, never by memory. `eval_rate` is the metric — see the
reproduction notes below for why.

**On the old `~3.2 tok/s` figure — the honest answer is that it cannot be attributed, and
two earlier attempts to settle it were both wrong.** The first claimed it was "wrong by
~3.4×" because it didn't match the 12B. The second claimed it was "correct all along"
because the 24B Q8 measures 3.26. Both inferred a model from a single matching number.
What the figure actually constrains is narrow but real: at ~80 GB·tok/s for dense weights
(below), **3.2 tok/s implies roughly 25 GB of dense weights** — which is satisfied by a 24B
at Q8 *or* a ~45B at Q4, and nothing on record distinguishes them. Record it as
"unattributed, implies ~25 GB dense" and stop there.

The lesson bites in both directions: an unqualified tok/s figure can be wrongly *trusted*
(the original error — cited as if it described this host generally) **and** wrongly
*dismissed* (both corrections). Always record the model, quant and date.

Note the 4B's `eval count` varies wildly between runs (1810 / 8094 / 2853 tokens) because
the model ignores the word limit and rambles — **`eval rate` is the stable metric, total
duration is not**. Compare rates, never wall-clock, when re-running this.

To reproduce, use the same prompt and model tags and report all three runs. If the spread
exceeds a few percent, something else is contending for the iGPU and the numbers are not
comparable.

## Bytes read per token is the lever — which means MoE, not size

The measurements span **3.26 → 24.82 tok/s, a 7.6× range, on identical hardware and
identical config.** The variable is **how many bytes of weights cross the memory bus per
token** — not parameter count, and not file size.

For **dense** models those are the same thing, so throughput tracks size inversely and the
product is near-constant:

| Dense model | Size | tok/s | size × tok/s |
|---|---|---|---|
| Qwen3.5-4B | 2.7 GB | 23.54 | 64 |
| Moonlit-Mirage-12B | 7.5 GB | 10.88 | 82 |
| MistralRP-Noromaid-7B | 7.7 GB | 10.43 | 80 |
| Qwen3.6-27B-Fable | 18 GB | 4.28 | 77 |
| Dolphin-Mistral-24B Q8 | 25 GB | 3.26 | 81 |

**For dense models on this box: `tok/s ≈ 80 ÷ size in GB`.** Accurate to a few percent
above ~7 GB (the 4B undershoots because fixed overheads dominate when the model is tiny).

**For MoE models that rule is meaningless**, because only the active experts are read per
token:

| MoE model | Size | Active | tok/s | Dense rule would predict |
|---|---|---|---|---|
| gemma-4-26B-A4B | 16 GB | ~4B | **24.82** | 5.0 |
| Melody1437-26B-A4B | 17 GB | ~4B | **19.71** | 4.7 |

Off by **5×**. An earlier revision of this section stated `80 ÷ GB` as a universal rule for
this host — it is dense-only, and every model measured at the time happened to be dense,
which is exactly why the error survived being "validated".

**The practical consequence, and it is the single most useful finding here:**

> The **fastest** model on this box is also one of the **largest**. `gemma-4-26B-A4B` at
> 16 GB runs at 24.82 tok/s — faster than the 7.5 GB dense 12B (10.88) and **5.8× faster
> than the 18 GB dense 27B** (4.28), which is nearly the same file size.

So: **total size decides whether it fits; active parameters decide how fast it runs.** When
choosing a model here, read the name for an `A<n>B` suffix (`26B-A4B` = 26B total, 4B
active) before reading the file size. On this hardware **MoE is not a nice-to-have, it is
the difference between usable and not** — and dense above ~12B is largely not viable.

Secondary, still worth acting on: **quantisation is cheap speed.** The 24B is a `Q8_0` at
3.26 tok/s; its `Q4_K_M` would be roughly half the bytes and therefore roughly **twice as
fast**, for a quality difference that is small at 24B.

Both of these dwarf the UMA question below, which is expected to be a single-digit-percent
difference either way.

# Tuning notes

* `OLLAMA_FLASH_ATTENTION=1` + `OLLAMA_KV_CACHE_TYPE=q8_0` only matter once the GPU
  backend is live (they did nothing on CPU). q8_0 halves KV cache, nearly lossless — but
  it is a **memory** lever only. A/B tested 2026-07-29 against f16 (see below): measured
  speed effect is ≈0%, so keep q8_0 for the freed memory headroom under the 16384 context,
  not for any throughput gain.
* `OLLAMA_KEEP_ALIVE=-1` pins models in memory (shows as `Forever` in `ollama ps` and a
  giant duration in logs — normal). With q8 cache + 16384 ctx on a shared-DDR5 iGPU, watch
  for memory pressure — it's the first suspect if loads crawl or OOM. This is why
  `OLLAMA_MAX_LOADED_MODELS` matters: pinning is indefinite, so the model cap is the only
  thing bounding total held memory.
* Discovery may set a huge `default_num_ctx` (262144, confirmed in the logs 2026-07-29)
  because it sees the full UMA+GTT pool. The explicit `OLLAMA_CONTEXT_LENGTH=16384`
  overrides per-model; keep it set.
* **CPU governor is already optimal and is not a lever here.** Confirmed 2026-07-29:
  all 24 threads report `scaling_governor=performance` and
  `energy_performance_preference=performance` under `amd-pstate-epp` with
  `amd_pstate status=active`. There is no `/sys/firmware/acpi/platform_profile` on this
  host. `/sys/class/platform-profile/` does exist but is empty (no registered devices) —
  no platform-profile handler is bound, so there's no ACPI platform-profile knob to pull
  either way; that interface is laptop firmware, not present on the N5 Pro under Proxmox.
  Community guidance ranking governor/power-profile as the top lever for this CPU was
  written against a machine that exposed it; here it is already done.
* **GPU `power_dpm_force_performance_level` (auto vs high) tested 2026-07-29 — reverted,
  not a lever.** Forced `card1` (Radeon 890M) from `auto` to `high` at runtime
  (`/sys/class/drm/card1/device/power_dpm_force_performance_level`, verified read-back both
  ways, no reboot needed) and ran the harness 3x per model against the
  `baseline-uma32-gtt31-ctx16k-kvq8` numbers measured earlier the same day. gemma-4-26B-A4B
  (MoE): baseline 24.82 tok/s vs dpm-high mean 24.96 tok/s (+0.56%). Moonlit-Mirage-12B
  (dense): baseline 10.88 tok/s vs dpm-high mean 10.98 tok/s (+0.92%). **Decision rule,
  fixed in advance:** both models >3% gain to keep `high` pinned; neither cleared it, so
  reverted to `auto` (confirmed read-back `auto`); pinning clocks for a <1% swing has no
  upside, and idle power is presumed (not measured) to be worse at `high`. The sub-1%
  deltas on both a MoE and a dense
  model corroborate the bandwidth-bound diagnosis in *UMA and GTT*, below: this iGPU is
  starved on memory bandwidth, not compute, so GPU core clock/power-state tuning is not
  worth pursuing further here.
* **KV cache type (f16 vs q8_0) A/B'd 2026-07-29 — reverted to q8_0, not a speed lever.**
  Backed up `override.conf` outside the drop-in dir (`/root/ollama-override.bak-task6`),
  switched `OLLAMA_KV_CACHE_TYPE` q8_0 → f16, `daemon-reload` + restart, and verified the
  **live process** (not the file) via `systemctl show ollama -p Environment` both before
  and after — the drop-in directory must stay clean of stray files, a past incident there
  silently broke env propagation. Ran the harness 3x per model against the
  `baseline-uma32-gtt31-ctx16k-kvq8` numbers (gemma-4-26B-A4B, MoE, 24.82 tok/s;
  Moonlit-Mirage-12B, dense, 10.88 tok/s). kv-f16 means: gemma-4-26B-A4B (MoE) 24.77 tok/s
  (**-0.20%**), Moonlit-Mirage-12B (dense) 11.00 tok/s (**+1.10%** — but this sits inside
  the 12B's own documented 1.2% run-to-run spread in the Baseline table, i.e. within noise
  per the harness section's comparability warning). **Decision rule, fixed in advance:**
  both models needed to gain more than 3% to keep f16 pinned; neither cleared it
  (gemma-4-26B-A4B actually regressed slightly), so reverted to q8_0 — confirmed read-back
  `OLLAMA_KV_CACHE_TYPE=q8_0` and `active` from the live process, drop-in directory clean,
  backup removed. The same community writeup cited in [3] ran a dedicated KV A/B on this
  model (f16 29.0 vs q8_0 29.0 at ctx 32K — a separate test from its 29.5 tok/s headline
  figure) and reports the "~10% faster" folklore as contradicted by every published
  measurement (-3% to 0%); q8_0's value here is the halved KV cache memory footprint, not
  throughput. Final state: `OLLAMA_KV_CACHE_TYPE=q8_0`, service active.
* **Software-currency check (Ollama + Mesa) 2026-07-29 — neither upgraded, both within
  acceptable margin.** Installed Ollama **0.31.1**, latest GitHub release **v0.32.5**
  (`api.github.com/repos/ollama/ollama/releases/latest`) — one minor version ahead, which
  is inside the "couple of minor versions" no-upgrade threshold set in advance, so the
  installer was not run and no snapshot was taken. Mesa: installed
  **25.0.7-2~bpo12+1**, `apt-cache policy mesa-vulkan-drivers` candidate is the **same**
  version — nothing ≥25.3 is available in bookworm-backports, so Mesa 25.3+ is not
  currently reachable on this host without pulling from trixie/testing, which is out of
  scope given the glibc/libdrm risk to the working Vulkan chain. Citation [3] measured a
  stale llama.cpp build at 56% slower and Mesa 25.3+ at +19.8% prefill on the same
  silicon (Unraid/llama.cpp stack, not this host) — neither figure was reproduced here
  since neither upgrade path was actionable; recorded as a cross-reference only.
  **Decision rule, fixed in advance:** upgrade only if the release is materially ahead
  (multiple minors or a major) or a genuinely newer backports Mesa exists — neither held,
  so this was a no-op by design, not a skipped check. No benchmarking run (nothing was
  upgraded), no snapshot exists post-check (`pct listsnapshot 102` shows only `current`).
  Final state: Ollama 0.31.1, Mesa 25.0.7-2~bpo12+1, both unchanged.

# UMA and GTT — the memory pool, and how it's split

Previously headed "the real performance lever: BIOS UMA frame buffer". That framing was
wrong twice over and has been corrected: **model architecture is the real lever** (7.6×,
see [above](#bytes-read-per-token-is-the-lever--which-means-moe-not-size)), and UMA is only
half of the memory story.

iGPU inference is **memory-bandwidth bound** on DDR5, so the ceiling is inherent, not a
misconfig. More CPU cores don't help (bandwidth, not compute, is the limit).

There are **two** pools the iGPU can draw on, and this doc previously only discussed one:

* **UMA** — the BIOS frame buffer. **Permanently stolen** from the OS: the host sees 62Gi
  of 96GB physical because 32GB is carved out before boot.
* **GTT** — system RAM the GPU **borrows and gives back**. Governed by the kernel's TTM
  `pages_limit`, not by BIOS.

Live state as of 2026-07-29:

```bash
cat /proc/cmdline                                              # no ttm.* params set
cat /sys/module/ttm/parameters/pages_limit                     # 8180431 pages = 31.2 GiB
cat /sys/class/drm/card1/device/mem_info_gtt_total             # 33507045376 = 31.2 GiB
```

So the iGPU addresses **32 GB UMA + 31.2 GiB GTT ≈ 63.2 GiB**, which is exactly what Ollama
reports (`total="63.2 GiB" available="63.0 GiB"`). The GTT half is at the kernel default —
**no `ttm.pages_limit` has ever been set on this host.**

**This inverts the advice this doc has been carrying.** The received wisdom for this exact
chip (Ryzen AI 9 HX 370 / gfx1150 / Strix Point) is **small UMA, large GTT** — 8–16GB UMA,
with `ttm.pages_limit` raised to cover the rest — because UMA is permanently lost while GTT
is only borrowed. Raising GTT is done at the bootloader, needs no BIOS trip, and is
reversible:

```
ttm.pages_limit=18874368 ttm.page_pool_size=18874368     # 72GB (pages × 4KiB)
```

Note `amdgpu.gttsize` is deprecated on current kernels (warns, then ignores you), and
`amdttm.pages_limit` is for the out-of-tree DKMS module. Verify by reading
`mem_info_gtt_total` back, never by trusting the config.

The counterweight: UMA measures roughly **+11% faster than GTT** for the same capacity, so
the trade is ~11% throughput against 16–24GB permanently returned to the fleet. That, not
"32 vs 24", is the real decision — see [actions.md](/actions.md).

The BIOS steps themselves, if changing UMA:

1. Shut down guests cleanly, reboot host, enter BIOS (`Del` or `F2`).
2. Path is firmware-dependent: `Advanced → AMD CBS → NBIO Common Options → GFX
   Configuration` (or `UMA Frame Buffer Size`), set UMA mode to
   `UMA_SPECIFIED`/`Manual`.
3. **Currently 32GB** (`dmesg | grep -i 'VRAM:'` → `VRAM: 32768M`, confirmed live
   2026-07-25 and again 2026-07-29 — this doc previously said "currently 2GB," which was
   stale). The old "24GB sweet spot" reasoning here was built on covering *a q4 30B dense
   model (~18GB) in UMA*, which the Baseline now shows is the wrong model class to design
   around: the 18GB dense 27B runs at 4.28 tok/s and is not a sensible daily driver at any
   UMA size, while the 16GB MoE that *is* worth running fits comfortably either way.
   **This doc previously claimed there was "no measured inference benefit over 24GB". That
   claim was never true and has been removed: 24GB has never been run.**
4. Save (`F10`), reboot. Verify on host: `dmesg | grep -i 'VRAM:'` reads back your target
   (e.g. `24576M` for 24GB), and `free -h` reflects the returned RAM.

## The A/B worth running

The originally-planned test was **32GB vs 24GB UMA**. That is now the *less* interesting
question — 8GB either way, on the pool that costs the fleet permanently, for an expected
single-digit-percent throughput difference.

The better test, given GTT has never been touched here:

* **A-side** — current: UMA 32GB, GTT 31.2 GiB (default), pool 63.2 GiB. Baseline figures
  above.
* **B-side** — UMA **16GB** in BIOS, plus `ttm.pages_limit=18874368` on the kernel command
  line. Returns **16GB permanently to the fleet** (host should show ~78Gi, not 62Gi) while
  *growing* the addressable pool.

Verify both halves before benchmarking — `dmesg | grep -i 'VRAM:'` for UMA,
`cat /sys/class/drm/card1/device/mem_info_gtt_total` for GTT, and Ollama's `total=` line
for the sum. Then re-run the exact Baseline prompt and models.

Decision rule, fixed in advance: **if the MoE (`gemma-4-26B-A4B`, the model actually worth
running) stays within ~10% of 24.82 tok/s, take the B-side** — 16GB back to the fleet is
worth a modest throughput cost. If it drops materially more than that, UMA is doing real
work and 32GB stands.

Note the GTT half can be tested **without a BIOS trip at all** — `ttm.pages_limit` is a
kernel parameter, so it needs a reboot but not console access. That half is separable and
much cheaper to try first.

# Citations

[1] n5-pro-homelab Claude Skill — references/gpu-ollama.md (Dave's claude.ai account, last updated 2026-07-19)
[2] direct host review 2026-07-29 — `pct exec 102 -- ollama run --verbose` benchmark runs (7 models, three runs each via `/root/bench-ollama.sh`), `systemctl show ollama -p Environment`, `ollama list`/`ollama ps`, `dmesg | grep VRAM:`, `/proc/cmdline`, `/sys/module/ttm/parameters/pages_limit`, `mem_info_gtt_total`, ollama journal `inference compute` line (Baseline, dense-vs-MoE finding, UMA/GTT split, live override contents, concurrency bounds)
[3] Community writeup, same silicon (Ryzen AI 9 HX 370 / Radeon 890M / gfx1150, 96GB DDR5-5600) on Unraid + llama.cpp/llama-swap rather than this host's Proxmox + Ollama — 13 models benchmarked. Source of the small-UMA/large-GTT guidance, the `ttm.pages_limit` syntax, the deprecation of `amdgpu.gttsize`, the ~+11% UMA-over-GTT figure, Vulkan-over-ROCm for gfx1150, the bytes-per-token model, Ollama bug #16462, the KV-cache f16-vs-q8_0 zero-effect finding and the -3%..0% published-measurement range for the "~10% faster" KV quantisation folklore, and the software-currency figures of a stale llama.cpp build measuring 56% slower and Mesa 25.3+ measuring +19.8% prefill. Its gemma-4-26B-A4B measured 29.5 tok/s against 24.82 here (main benchmark table, single-stream) — different stack, same regime; its separate dedicated KV-cache A/B table for the same model (f16 29.0 vs q8_0 29.0 tok/s, ctx 32K) is a different test run from that 29.5 headline figure. Treat as cross-reference, not as a description of this host.
