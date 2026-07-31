---
type: Playbook
title: GPU passthrough & Ollama on Vulkan (CT102)
description: The full iGPU → Ollama-on-Vulkan chain for CT102 — device passthrough, idmap, Mesa backports, and the IGPU_ENABLE flag.
tags: [gpu, ollama, vulkan, lxc, proxmox]
timestamp: 2026-07-30T00:00:00Z
---

The Radeon 890M (RDNA 3.5, `gfx1150`, reported as `GFX1150`/`0x150e`) on
[the N5 Pro host](/host/n5-pro.md) drives [ollama (CT102)](/containers/ollama.md) via the
**Vulkan** backend (RADV), not ROCm — ROCm's gfx1150 support is flaky; Vulkan is the
working path here. This chain was fought end-to-end and every link matters. Symptoms that
send you here: `library=cpu`, `total_vram="0 B"`, `dropping integrated GPU`, or slow
inference.

**This page is about making the GPU work at all. For making it *fast*, see
[Local LLM daily driver](local-llm-daily-driver.md)** — model shape (the single biggest
lever, and one this page previously got wrong), the GTT ceiling that may be capping loads
at ~31 GiB, the CPU governor, and the ZFS-ARC memory contention that can hard-lock the
host.

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

**Which 25.x you land on matters, and this install is unpinned.** Mesa **25.3+** carries
Valve's RADV CU-mode/LDS patches, externally measured at **+19.8% prefill** on this GPU
family. Backports moves, so check what actually landed rather than assuming — add
`driverInfo` to the `grep` above.

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
`inference compute ... library=Vulkan ... type=iGPU total="104.0 GiB"` (measured 2026-07-29
after the GTT raise; it read `63.2 GiB` earlier that same day at the kernel-default GTT, and
this doc before that said `48.0 GiB`). The figure is **UMA + GTT** — 32 GB frame buffer plus
72 GiB of borrowable system RAM — so capacity is NOT capped at the UMA carve-out. See
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

Secondary, acted on: **quantisation is cheap speed — tested, not just predicted.** The 24B
`Q8_0` ran at 3.26 tok/s (3.22 post-`gtt72`); a `Q4_K_M` was pulled alongside it 2026-07-29
(14 GB on disk) and measured at **5.63 tok/s** — **1.75×** the Q8, not quite the "roughly
twice as fast" the halved-bytes prediction implied, but in line with the dense rule's
`~80 ÷ size-in-GB` (14 GB × 5.63 = 79). Decided: the owner kept the `Q4_K_M` and removed
the `Q8_0` (2026-07-29) — the quality difference wasn't judged worth the Q8's slower
throughput. Details and the full model-inventory table:
[containers/ollama.md](/containers/ollama.md#model-inventory-2026-07-29).

Both of these dwarf the UMA question below, which is expected to be a single-digit-percent
difference either way.

**Do not read that 48 GiB as the real ceiling.** It is Ollama's own estimate of what it
believes it can address, not the kernel's limit. The TTM `pages_limit` default is **half
of system RAM**, which on this host (62GB visible, after the 32GB UMA carve-out) works out
to **~31 GiB** — so loads above that may fail regardless of what Ollama advertises.
Unverified; check with `cat /sys/class/drm/card1/device/mem_info_gtt_total` and see
[Local LLM daily driver](local-llm-daily-driver.md#the-gtt-ceiling--likely-capped-at-31-gib-here)
for how to raise it (and why `/etc/default/grub` may be a dead file on this host).

# Tuning notes

* `OLLAMA_FLASH_ATTENTION=1` + `OLLAMA_KV_CACHE_TYPE=q8_0` only matter once the GPU
  backend is live (they did nothing on CPU). q8_0 halves KV cache, nearly lossless — worth
  it with the 12400 context.
* `OLLAMA_KEEP_ALIVE=-1` pins models in memory (shows as a giant duration in logs —
  normal). With q8 cache + 12400 ctx on a shared-DDR5 iGPU, watch for memory pressure —
  it's the first suspect if loads crawl or OOM. **Stronger warning added 2026-07-28:** on
  unified memory a GPU OOM can **hard-lock the whole host**, taking the NAS and all LAN
  DNS with it, and ZFS ARC is competing for the same RAM the iGPU borrows as GTT. Pinning
  indefinitely is the wrong default while ARC is also expanding, and CT102 has no memory
  limit to backstop it. See
  [Local LLM daily driver](local-llm-daily-driver.md#memory-contention--the-risk-that-matters-more-than-any-speed-number).
* `OLLAMA_KV_CACHE_TYPE=q8_0` is a **memory** lever, not a speed one — externally measured
  flat within noise (29.0 t/s either way) while halving the KV cache. The value is context
  headroom. Both preconditions are already met here: `-fa on` (via
  `OLLAMA_FLASH_ATTENTION=1`) and k/v matching, which this variable sets together.
* Discovery may set a huge `default_num_ctx` (e.g. 262144) because it sees ~48GB. The
  explicit `OLLAMA_CONTEXT_LENGTH=12400` overrides per-model; keep it set.

**Decision rule, fixed in advance:** GTT kept regardless (its value is capacity), but a >3%
drop on any model would have meant something *else* changed in the reboot and demanded
investigation. Every model came in within 2.1% of Baseline, inside the run-to-run noise
band. Final state: `ttm.pages_limit=18874368`, pool 104 GiB, throughput unchanged.

Measured generation was ~3.2 tok/s on shared memory — iGPU inference is
**memory-bandwidth bound** on DDR5. More CPU cores don't help (bandwidth, not compute, is
the limit).

**The "so this is inherent, not a misconfig" conclusion that used to follow was wrong, and
BIOS UMA is not "the one hardware lever" — corrected 2026-07-28.** Bandwidth-bound is
right; treating bytes-per-token as fixed is not. ~3.2 tok/s is exactly what the bandwidth
model predicts for a **~30B dense q4** model (~18 GB/token ÷ ~60 GB/s), so the chain is
performing to spec — but a ~30B **MoE** with ~3B active reads ~2 GB/token and should run
roughly **7× faster** on the same hardware. Model shape is the biggest lever here, and it
costs a model pull rather than a reboot. See
[Local LLM daily driver](local-llm-daily-driver.md#the-one-law-bytes-per-token-not-parameter-count).

The BIOS UMA lever below is still real, but it is worth ~11%, not 7×:

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

The better test:

* **A-side** — UMA 32GB, GTT 31.2 GiB (default), pool 63.2 GiB. Baseline figures above.
  (Live state until 2026-07-29.)
* **B-side** — UMA **16GB** in BIOS. That is the only change left to make: GTT is already
  72 GiB on both sides since 2026-07-29, so the B-side needs no kernel-command-line work.
  Returns **16GB permanently to the fleet** (host should show ~78Gi, not 62Gi) while the
  pool stays large (16 + 72 ≈ 88 GiB).

Verify both halves before benchmarking — `dmesg | grep -i 'VRAM:'` for UMA,
`cat /sys/class/drm/card1/device/mem_info_gtt_total` for GTT (should still read 72 GiB),
and Ollama's `total=` line for the sum. Then re-run the exact Baseline prompt and models.

Decision rule, fixed in advance: **if the MoE (`gemma-4-26B-A4B`, the model actually worth
running) stays within ~10% of 24.49 tok/s — i.e. above ~22.0 — take the B-side** (the
comparison base is the new A-side's `gtt72` figure, not the pre-GTT Baseline). 16GB back to
the fleet is worth a modest throughput cost. If it drops materially more than that, UMA is
doing real work and 32GB stands.

The GTT half was indeed separable and cheaper — no BIOS trip, just a reboot — and it was
applied on its own 2026-07-29 and measured flat (see the `gtt72` table above). The state
now running (UMA 32GB + GTT 72 GiB, pool 104 GiB) is the new A-side; **only the UMA
reduction is still untested.**

# Citations

[1] n5-pro-homelab Claude Skill — references/gpu-ollama.md (Dave's claude.ai account, last updated 2026-07-19)
[2] direct host review 2026-07-29 — `pct exec 102 -- ollama run --verbose` benchmark runs (7 models, three runs each via `/root/bench-ollama.sh`), `systemctl show ollama -p Environment`, `ollama list`/`ollama ps`, `dmesg | grep VRAM:`, `/proc/cmdline`, `/sys/module/ttm/parameters/pages_limit`, `mem_info_gtt_total`, ollama journal `inference compute` line (Baseline, dense-vs-MoE finding, UMA/GTT split, live override contents, concurrency bounds); same-day GTT raise applied and verified live — `/etc/default/grub` edit + `update-grub` + fleet reboot, post-reboot `/proc/cmdline`, `pages_limit` 18874368, `mem_info_gtt_total` 77309411328, `total="104.0 GiB"` journal line, the CT102 cgroup limit (`memory: 49152`) confirmed in `/etc/pve/lxc/102.conf` before the reboot, the mid-shutdown LAN DNS outage (AdGuard down with the fleet, severing the driving session), and the `gtt72` 7-model benchmark run
[3] Community writeup, same silicon (Ryzen AI 9 HX 370 / Radeon 890M / gfx1150, 96GB DDR5-5600) on Unraid + llama.cpp/llama-swap rather than this host's Proxmox + Ollama — 13 models benchmarked. Source of the small-UMA/large-GTT guidance, the `ttm.pages_limit` syntax, the deprecation of `amdgpu.gttsize`, the ~+11% UMA-over-GTT figure, Vulkan-over-ROCm for gfx1150, the bytes-per-token model, Ollama bug #16462, the KV-cache f16-vs-q8_0 zero-effect finding and the -3%..0% published-measurement range for the "~10% faster" KV quantisation folklore, and the software-currency figures of a stale llama.cpp build measuring 56% slower and Mesa 25.3+ measuring +19.8% prefill. Its gemma-4-26B-A4B measured 29.5 tok/s against 24.82 here (main benchmark table, single-stream) — different stack, same regime; its separate dedicated KV-cache A/B table for the same model (f16 29.0 vs q8_0 29.0 tok/s, ctx 32K) is a different test run from that 29.5 headline figure. Treat as cross-reference, not as a description of this host.
