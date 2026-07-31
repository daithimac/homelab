---
type: Playbook
title: Local LLM daily driver — tuning the 890M for real inference speed
description: How to get usable tokens/sec out of the Radeon 890M — model shape, the GTT ceiling, the UMA trade, and the memory-contention risk unique to this NAS.
tags: [ollama, llm, vulkan, gpu, performance, tuning, gtt, moe]
timestamp: 2026-07-28T00:00:00Z
---

**Everything on this page is external and unverified on this host.** It comes from a
July 2026 write-up by another owner of the *same* silicon — Ryzen AI 9 HX 370, Radeon
890M, `gfx1150`, 96GB dual-channel SO-DIMM DDR5 — who benchmarked 13 models and published
their flags and mistakes [1]. Their numbers are not measurements of this box. They are
predictions for it, and good ones, because the hardware matches. Run the checks in
[Verification sequence](#verification-sequence) before treating any figure here as fact
about this machine.

Their platform differs in two ways that change *how* you apply things, never *whether*:
they run **Unraid with bare-metal Docker**, this host runs **Proxmox with an unprivileged
LXC** ([ollama (CT102)](/containers/ollama.md)). The bootloader and cgroup details below
are translated accordingly.

For the passthrough chain itself — device entries, idmap, Mesa, `OLLAMA_IGPU_ENABLE=1` —
see [GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md). That chain is
working. This page is about what to do once it is.

# The one law: bytes per token, not parameter count

Generation speed on this box is set by **how many bytes must be read from RAM per token**,
not by how large the model is. Dual-channel DDR5-5600 gives ~89.6 GB/s on paper and
60–67 GB/s in practice, and that is the ceiling on everything.

The consequence is counterintuitive and it is the single most useful thing on this page:

* A **117B MoE** with ~5B active reads ~2.7 GB/token and was measured at **20.7 t/s**.
* A **12B dense** model reads ~6 GB/token and was measured at **10.6 t/s**.

The 117B model is twice as fast as the 12B one. **Total parameters decide whether it
fits; active parameters decide how fast it runs.**

MoE decode only achieves 56–61% of theoretical bandwidth (versus 79–83% for dense),
because `mul_mat_id` gathers non-contiguous expert rows. So when predicting: compute
bytes/token ÷ effective bandwidth, then **multiply MoE predictions by 0.6**.

## This explains the 3.2 tok/s already recorded here

[GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md) records ~3.2 tok/s
and attributes it to DDR5 bandwidth being "inherent, not a misconfig." Half right — it *is*
bandwidth-bound, but the doc treats bytes/token as fixed when it is the one variable fully
under our control:

| Model shape | Bytes/token | Predicted | 
|---|---|---|
| 7B dense q4 | ~4 GB | ~15 t/s |
| 14B dense q4 | ~8 GB | ~7.5 t/s |
| **30B dense q4** | **~18 GB** | **~3.3 t/s** |
| 35B-A3B MoE q4 | ~2 GB | ~21 t/s |

The recorded 3.2 tok/s lands on the 30B-dense prediction to within noise. Read that as
**good news**: the passthrough chain is performing to spec and the hardware is healthy.
The model shape is the problem, not the configuration.

**And this is not hypothetical here.** [sillytavern (CT120)](/containers/sillytavern.md)'s
configured chat model is `Moonlit-Mirage-12B-i1-GGUF` — a **12B dense** model, the exact
shape the source author used as their *dense control* and measured at 10.6 t/s against
21.6 for a 35B MoE. At least one daily-driver frontend on this box is therefore running
the worst-performing model shape available to it, confirmed from the bundle rather than
inferred. (Their roleplay model choice is a judgement call, not just a speed one — an MoE
equivalent needs finding rather than assuming.)

**So: MoE only.** Dense above ~12B is not viable at this bandwidth. A ~30B-class MoE with
~3B active should give roughly **7× the current speed** at comparable or better quality,
for the cost of a model pull and no reboot. This supersedes the "7–14B q4 is the pleasant
range" and "a 30B model will still be slow" guidance on the passthrough page, both of which
are true for dense models and false for MoE.

# Vulkan over ROCm — why the existing choice is right

The passthrough playbook already chose Vulkan on the grounds that "ROCm's gfx1150 support
is flaky." The external write-up supplies the concrete mechanism, which is worth recording
because it is much stronger than "flaky":

**ROCm on gfx1150 can only allocate inside the BIOS UMA carve-out.** `hipMallocManaged` is
unsupported (llamacpp-rocm #57, open, assigned to AMD). Reserve 4GB in BIOS and ROCm sees
4GB — the rest of the system RAM is invisible to it. Vulkan addresses GTT instead, so it
reaches the whole pool. Their measured split on Qwen3-8B:

| Backend | Prefill t/s | Generation t/s |
|---|---|---|
| Vulkan (RADV) | 146 | **9.87** |
| ROCm (HIP) | **207** | 4.76 |
| CPU (24 threads) | 132 | 2.59 |

ROCm wins prefill, Vulkan wins generation, and generation is what you feel. They note a
separate run put the generation gap far closer (14.11 vs 12.73), so don't bank the exact
multiple — it doesn't change the answer, because ROCm seeing only the carve-out is
disqualifying regardless of its prefill advantage.

Note also that **the advice circulating online to prefer ROCm on AMD is written for
gfx1151 (Strix Halo)**, a different chip with ~4× the memory bandwidth. Treat any
impressive "AMD unified memory LLM" number found online as Halo unless proven otherwise.

Two configuration points this confirms about the current setup, both already correct here:
pass **`/dev/dri` and nothing else** (`/dev/kfd` is ROCm-only, and 102.conf does not have
it), and **no `HSA_OVERRIDE_GFX_VERSION` anywhere**.

# The GTT ceiling — likely capped at ~31 GiB here

GTT is the pool of system RAM the GPU is allowed to borrow. On a box with no real VRAM it
is the thing that decides what will load, and **the kernel's TTM `pages_limit` defaults to
half of system RAM**, silently.

Applied to this host: 96GB physical − 32GB UMA carve-out = **62GB visible to the OS**
(confirmed by `free -h`; see [n5-pro](/host/n5-pro.md)). Half of 62 is **~31 GiB of GTT**.

This matters because the passthrough playbook records the success condition as Ollama
reporting `total="48.0 GiB"`. **That is Ollama's own estimate of what it believes it can
address, not the kernel's actual ceiling.** If the arithmetic holds, anything over ~31 GiB
fails to load no matter what Ollama advertises. Unverified — check it:

```bash
cat /sys/class/drm/card1/device/mem_info_gtt_total
```

Note **`card1`**, not `card0` as in the source write-up — the iGPU on this host is `card1`
(`226:1`), per the passthrough playbook.

## Raising it — and the Proxmox version of their week-long mistake

The kernel parameters, sized to 72GB (pages × 4KiB):

```
ttm.pages_limit=18874368 ttm.page_pool_size=18874368
```

`16777216` would be 64GB. **Do not** use `amdgpu.gttsize` — deprecated on current kernels,
it warns and then ignores you. **Do not** use `amdttm.pages_limit` either, which is for the
out-of-tree DKMS module.

The author lost a week to these parameters being set in a file that was no longer read:
Unraid 7.3 had migrated boot to GRUB, leaving `syslinux.cfg` a dead leftover, and the
params **never once applied** while appearing correct in the config they were reading.

**The identical trap exists here in different clothing.** If this host boots via
`proxmox-boot-tool`/systemd-boot — standard on ZFS root — then editing `/etc/default/grub`
does *nothing at all*. The live path is `/etc/kernel/cmdline` followed by
`proxmox-boot-tool refresh`. Establish which applies before touching anything, and back up
the boot config first:

```bash
proxmox-boot-tool status
cat /proc/cmdline
```

Then verify against `mem_info_gtt_total` after the reboot rather than trusting the config
you just edited — and **re-check it after every `pve-kernel` upgrade**, which is this
host's equivalent of their "re-check after every Unraid update."

# The UMA trade, now quantified

The standing decision in [actions.md](/actions.md) frames the BIOS UMA frame buffer as
32GB (current) versus a documented 24GB sweet spot. The external write-up reframes it and,
for the first time, puts a number on the cost of each side:

* Their recommendation is **UMA small (8–16GB), GTT large**, on the principle that **UMA is
  permanently stolen from the system while GTT is borrowed and given back**.
* But UMA-resident inference measured **~+11%** over GTT (21.33 vs 19.18 t/s on a 35B MoE).

So the trade is now numeric: the current 32GB buys roughly 11% on whatever fits inside it,
and charges every other guest 8GB permanently. **On this host that trade is worse than on
theirs** — they run one Immich instance alongside, this box runs fourteen other guests plus
NAS, DNS and smart-home duty.

If this gets actioned, the argument now points at **16GB rather than 24GB**, with GTT
raised to carry the model pool. A second-order benefit makes it more attractive than when
the decision was parked: dropping UMA to 16GB raises OS-visible RAM to 80GB, which lifts
the *default* GTT ceiling to ~40 GiB before the kernel parameter is even set.

Still requires a reboot into BIOS with Dave at the console. See the UMA section of
[GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md#the-real-performance-lever-bios-uma-frame-buffer)
for the BIOS path.

# The lever nobody here has pulled: CPU governor

Ranked **first** in their list of untried levers, and measured on this exact CPU:
**+36% generation and +120% prefill** from the platform power profile / CPU governor.

They are explicit that this is *not* the same knob as GPU perf-mode, which gave +1% —
noise, and itself a useful result, since it confirms generation is bandwidth-bound rather
than compute-bound. The governor moves the **memory controller clock**, which is precisely
the bottleneck.

Proxmox typically lands on `powersave` under `amd_pstate`. Free to test, instantly
reversible, no reboot:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cpupower frequency-set -g performance
```

Caveat specific to this host: it is also the NAS and the only DNS server, running
24/7. "Performance governor permanently" is a power and thermal decision, not a free win.
For *measuring*, it costs nothing.

# Memory contention — the risk that matters more than any speed number

Buried in their write-up: **on unified memory a GPU OOM hard-locks the entire machine.**
Their mitigation is a cgroup memory limit on the inference container, which converts a
whole-box lock into one dead container.

**The blast radius here is far worse than on their box.** A hard lock takes down the NAS,
[AdGuard](/containers/adguard.md) — *all* LAN DNS — Home Assistant, Jellyfin, and the other
ten guests at once. Three factors compound it, none of them previously documented:

* **ZFS ARC competes for the same RAM the iGPU wants as GTT.** They run one pool and still
  had to cap `zfs_arc_max` from 19.2 GiB down to 8 GiB just to load a 57 GiB model. This
  host runs **three pools** doing real NAS work — see [Storage](/storage/index.md).
* **`OLLAMA_KEEP_ALIVE=-1` pins models in RAM indefinitely.** The passthrough playbook
  already hedges ("watch for memory pressure"), but pinning is exactly the wrong default
  while ARC is also expanding. Their equivalent setting is a 15-minute idle unload.
* **There is no memory limit on CT102**, so nothing backstops a runaway allocation.

Their sequence for freeing room before a large load, translated to this host, stops the
competing consumers and caps ARC temporarily:

```bash
# check current ARC ceiling first
cat /sys/module/zfs/parameters/zfs_arc_max
echo 8589934592 > /sys/module/zfs/parameters/zfs_arc_max   # 8 GiB, temporary
# ... load and use the model ...
echo <original value> > /sys/module/zfs/parameters/zfs_arc_max
```

Setting a memory limit on CT102 is cheap insurance and should come **before** any
experiment with larger models, not after.

# What the models actually do

Selected rows from their 13-model benchmark — single stream, 200-token generations,
`cache_prompt:false`, ctx 32K unless noted, model confirmed via `/props` each run. Their
box, not this one:

| Model | Active | Quant | Actual t/s | Notes |
|---|---|---|---|---|
| gemma-4-26B-A4B | ~4B MoE | UD-Q4_K_XL | **29.5** | Fastest on the box |
| gpt-oss-20B | ~3.6B MoE | UD-Q4_K_XL | **28.7** | Effectively tied |
| Qwen3.6-35B-A3B-MTP | ~3B MoE | Q8_0 (36GB) | **22.0–22.5** | Their daily driver |
| Qwen3.6-35B-A3B | ~3B MoE | UD-Q4_K_XL (21.7GB) | **21.6** | Fits under a 31 GiB GTT ceiling |
| Qwen3-Next-80B-A3B-Thinking | ~3B MoE | UD-Q4_K_XL (43GB) | **22.4** | |
| gpt-oss-120B | ~5B MoE | UD-Q4_K_XL (59 GiB) | **20.7** | ctx 16K, 57.2 GiB GTT |
| Qwen3-Coder-Next 80B-A3B | ~3B MoE | UD-Q4_K_XL (49.6GB) | **17.9** | Best coder tested |
| gemma-4-12B | 12B **dense** | UD-Q4_K_XL | **10.6** | The dense control |

**Only the smaller entries are reachable here today.** Under a ~31 GiB GTT ceiling the
21.7GB Q4 35B-A3B and the 26–27GB gemma-class models fit; the 36GB Q8, 43GB, 49.6GB and
59GB entries do not, until GTT is raised.

Two findings that shape quant choice on RDNA 3.x specifically:

* **Q8 is cheaper than expected here.** A 36GB Q8 ran 19.4 t/s against a 21GB Q4's 21.6 —
  far less penalty than 2× the bytes implies.
* **IQ4_XS is marginally *faster* than K-quants** (Qwen3-30B-A3B: IQ4_XS 100.04 vs Q4_K_S
  98.51), so quant family is not a speed lever worth agonising over.

## Reasoning models feel slower than their token rate

Their daily driver thinks by default, and the overhead is a **floor rather than a
proportion**: a one-sentence answer took 48 tokens in and **489 out**, ~90% of it
reasoning. At 22 t/s that is 22 seconds for one sentence; a code-review prompt gave
"Thought for 3 minutes" before the first visible word.

| Question type | Real wait |
|---|---|
| One-line factual | 20–25s |
| Normal chat answer | 40–60s |
| Deep analysis | ~3 min |

Their conclusion is **rotation, not one model**: keep a fast non-reasoning model one config
line away for quick lookups, rather than disabling thinking on the reasoning model — the
benchmark scores that justify picking it are all measured with reasoning *on*, and turning
it off leaves something no better than a model that is already 33% faster.

Worth noting against this host's setup: Ollama's environment variables are **global**, so
expressing "fast model for lookups, slow model for analysis" means either per-request
overrides from the client or a different serving layer (see below).

# KV cache: a memory lever, not a speed lever

The passthrough playbook sets `OLLAMA_KV_CACHE_TYPE=q8_0`. Their measurements confirm this
is sound and clarify *why* — on gemma-4-26B-A4B at ctx 32K:

| KV type | gen t/s | GTT |
|---|---|---|
| f16 | 29.0 | 14,739 MiB |
| q8_0 | 29.0 | **14,019 MiB** |

Flat within noise, KV cache halved. Advice found elsewhere claiming ~10% faster is wrong;
every published measurement puts it between -3% and 0%. **The value is context headroom**,
which is what lets larger models go past 16K. Two preconditions, both already satisfied
here: `-fa on` (this host has `OLLAMA_FLASH_ATTENTION=1`) and **k and v must match** or AMD
drops off the fused flash-attention path (`OLLAMA_KV_CACHE_TYPE` sets both).

**KV cost varies ~10× by architecture** — `qwen35moe` (hybrid SSM, 10 of 41 layers
attention) is 20 KiB/token, `qwen3moe` ~96 KiB/token, `glm4moe` (47 attention layers)
188 KiB/token. Check before sizing context. The current `OLLAMA_CONTEXT_LENGTH=12400` was
chosen against a dense model; on a hybrid-attention MoE, 32K is likely affordable.

**A trap to know before it bites**, if speculative decoding is ever added here: they ran
`--spec-type draft-mtp` together with `--cache-type-k/v q8_0` for weeks, and **q8 KV gives
0% draft acceptance**. Speculation was slower than none while they believed it was giving
+16%. Two optimisations that each measure positively in isolation cancelled each other, and
nothing warned them. Ollama does not expose MTP today, so this is forward-looking only.

# Ollama may itself be costing something

They rejected Ollama outright, for reasons worth weighing rather than acting on
immediately:

* A **separate blob store** that re-downloads its own copies of weights.
* Bug **#16462**, where containerised Strix reports **2.0 GiB VRAM**.

That second one deserves a raised eyebrow here. The passthrough playbook *used* to state
the UMA frame buffer was "currently 2GB" and was corrected to 32GB on 2026-07-25. `dmesg`
confirmed `VRAM: 32768M`, so the BIOS setting is genuinely 32GB — but it is plausible the
stale "2GB" line originated as this Ollama bug being misread as a BIOS value rather than as
someone changing BIOS without updating the doc. Cheap to dismiss, worth knowing.

**No recommendation to replace Ollama.** It works, and both
[openwebui (CT106)](/containers/openwebui.md) and
[sillytavern (CT120)](/containers/sillytavern.md) point at `192.168.0.13:11434`. But their
stack — `llama-swap` in front of `llama.cpp`/Vulkan, loading and unloading on demand —
gives **per-model flags and TTL-based unloading** that Ollama's global-environment-variable
model cannot express. That matters exactly when rotating between a fast small model and a
slow large one, which is their central recommendation. If model rotation becomes the way
this box is used, revisit.

Their `llama-swap` operational notes, kept for that eventuality: bind-mount the **config
directory, not the file** (a single-file bind mount pins an inode, so any editor that
writes-and-renames leaves the container reading a ghost file forever — the same
`sed -i`-through-a-bind-mount trap that cost them three wrong theories); and llama-swap
**silently discards unknown config keys**, so every typo is invisible.

# Other levers, ranked by their measurements

Beyond model shape and the governor, in their order:

1. **`-b` / `-ub` batch-size sweep.** RADV picks matmul tile sizes at hard thresholds, so
   prefill falls off cliffs (llama.cpp #13765: a cliff at 385 tokens for Q4_K_M, 202 for
   Q4_K_S). `-b 256` alone took a 30B-A3B's prefill from 70 to 118 t/s elsewhere.
2. **Mesa 25.3+.** Valve's RADV CU-mode/LDS patches measured **+19.8% prefill**. The
   passthrough playbook installs "Mesa 25.x" from `bookworm-backports` **unpinned**, so
   what actually landed is unknown — check it.
3. **Build recency.** A stale `llama.cpp` build measured **56% slower** on the same model;
   two relevant PRs landed early 2026. Ollama bundles its own `llama.cpp`, so this
   translates to keeping the Ollama version current.
4. **Draft models / speculative decoding.** Dead on A3B MoEs — per-token cost is already so
   low there is little waste to recover — but Strix Halo took a 122B-**A10B** from 24.7 to
   35.3 t/s (+40%), and Qwen3-8B with a 0.6B draft measured +64–82% on this chip. Higher
   active-parameter counts are where it pays.

Two dead ends, recorded so nobody re-walks them: **extra CPU threads** (bandwidth-bound, no
effect) and **eGPU** (unsupported on Strix Point). The **NPU** is also a dead end for
serving — FastFlowLM is NPU-only rather than hybrid, hybrid prefill/decode is
Windows-exclusive, and it supports dozens of models against thousands of GGUFs — though it
does hit 60 t/s on Llama 3.2 1B, so the silicon works and there is simply no serving path.

# Model store growth

Their weights directory went **15GB → 280GB in three weeks**.

`AIVault/ollama-models` currently holds 90.6G against a hard **164G ceiling** — see
[AIVault](/storage/aivault.md#the-pool-is-62-full-but-nearly-all-of-it-is-a-reservation-found-2026-07-26),
where a thickly-provisioned 200G volume reserves the rest of the pool to store 19.7M.

The MoE models in the table above are 36GB, 43GB, 49.6GB, 57.2GB and 59GB each. **Two or
three of them exhausts the remaining headroom.** The AIVault reservation decision in
[actions.md](/actions.md) has been sitting as "decide eventually"; acting on the model-shape
finding above turns it into a blocker within days.

Their other storage practice is already satisfied here by a different route — they insist
weights go on NVMe outside the backup set, and AIVault *is* NVMe, while the 128GB boot disk
could not hold a model store regardless.

# Frontend gotchas

For [sillytavern (CT120)](/containers/sillytavern.md), two findings that hold regardless of
backend:

* **Settings are read once at page load.** An open tab keeps generating at old values and
  overwrites new ones on its next save. Hard-refresh every tab after any change — this
  silently clobbered an hour of their preset work.
* **Temperature 0.65 caused measurable mode collapse** — one probe turn, three swipes, all
  three sharing a frame and two word-for-word identical. At **temp 1.0** the lock broke.
  Their recommendation is temp 1.0, top_p 0.95, no repetition penalty.

Also worth knowing before tuning anything there: SillyTavern's Chat Completion panel for a
**Custom** source only exposes Temperature, Top P, Frequency Penalty and Presence Penalty.
`top_k`, `min_p` and repetition penalty are **never sent** — they had recorded values for
all three and credited a repetition-penalty fix that had never run, because the values they
"set" happened to match llama.cpp's defaults and therefore looked correct.

For [openwebui (CT106)](/containers/openwebui.md): reasoning output comes back in
`reasoning_content` and renders collapsed behind "Thinking…", so a reasoning model looks
exactly like a hang for anywhere from 20 seconds to 3 minutes. It isn't.

# A security finding that applies directly to this host

Their write-up records getting this wrong twice in one session. One item transfers exactly:

> "No login prompt appeared in my browser" is not evidence a service is safe. Your browser
> carries session state. Probe from off-box.

When they did, **Open WebUI returned 200 with no auth at all**.

This host has [openwebui (CT106)](/containers/openwebui.md) on `192.168.0.17:8080` and
**no active Proxmox firewall anywhere** (confirmed 2026-07-25, see
[n5-pro](/host/n5-pro.md)) — the LAN boundary is the entire security model, so an
unauthenticated service is exposed to every device on the LAN. Worth an actual off-box,
fresh-session probe rather than an assumption. Logged in [actions.md](/actions.md).

Their second item — never print a secret to stdout, rotate in place and verify by
behaviour rather than by echoing the value — is directly on point for the open P1 there
about plaintext credentials on docker-stack.

# Verification sequence

Nothing above is confirmed on this host. **The full ordered procedure now lives in
[AI optimisation runbook](ai-optimisation-runbook.md)** — baseline benchmark, memory
guardrail, model swap, cheap levers, reboot batch, with result tables to fill in and
rollback for each step. What follows is the short version.

In the order that gives the most information for the least risk:

```bash
# 1. Is GTT actually capped at ~31 GiB?  (card1, not card0)
cat /sys/class/drm/card1/device/mem_info_gtt_total

# 2. What did Mesa actually land on from backports?  Want 25.3+
pct exec 102 -- su -s /bin/bash ollama -c 'vulkaninfo --summary' | grep -iE 'driverInfo|driverName'

# 3. Which bootloader is live?  Decides where kernel params go
proxmox-boot-tool status
cat /proc/cmdline

# 4. What is ARC currently allowed to take?
cat /sys/module/zfs/parameters/zfs_arc_max

# 5. What governor is the host on?
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

Then, in increasing order of disruption:

1. **Pull a ~30B-class A3B MoE at q4 and measure.** Biggest single win, no reboot, no risk,
   and it either confirms or refutes the whole bytes-per-token model on this hardware.
2. **Test the CPU governor.** Free and instantly reversible.
3. **Set a memory limit on CT102** before experimenting with anything larger.
4. **Then the reboot-required pair together**: `ttm.pages_limit` via whichever bootloader
   path step 3 identified, and the BIOS UMA change — both need a host reboot, so batch them.

# Citations

[1] "96GB Ryzen AI 9 HX 370 on Minisforum N5 Pro as a daily-driver local LLM box: 13 models
benchmarked, every flag, and everything I got wrong" — r/MINISFORUM, July 2026. Supplied by
Dave 2026-07-28. External and unverified against this host; the author's box runs Unraid,
not Proxmox.

[2] Existing bundle state cross-referenced against [1]:
[GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md),
[ollama (CT102)](/containers/ollama.md), [AIVault](/storage/aivault.md),
[n5-pro](/host/n5-pro.md), [actions.md](/actions.md).
