---
type: Playbook
title: AI optimisation runbook — measuring and fixing local inference
description: Ordered, executable procedure to take the 890M from ~3.2 tok/s to its real ceiling, verifying each of the unverified claims in the daily-driver playbook as it goes.
tags: [ollama, llm, performance, tuning, runbook, gtt, moe, benchmark]
timestamp: 2026-07-29T00:00:00Z
---

The companion to [Local LLM daily driver](local-llm-daily-driver.md). That page holds the
*reasoning* — why bytes-per-token governs speed, where the numbers came from, what the
external source measured. This page is the *procedure*: run it top to bottom, record what
comes back, and it either confirms that page or refutes it.

**Nothing on that page has been verified on this host.** This runbook exists to change
that. Every phase ends with a number, and the bundle should only be updated with values
actually observed — the source author's central lesson is that unverified config beliefs
are the expensive kind.

**Ordering principle: safety before speed, and one change per measurement.** The single
most expensive mistake in the source material was two optimisations that each measured
positively in isolation and silently cancelled each other for weeks.

# Before you start

Phases 0–3 need no reboot and are individually reversible. Phase 4 needs a full host reboot
with someone at the console. Phase 5 is documentation.

This host is the NAS and the **only** DNS server for the LAN — see
[AdGuard](/containers/adguard.md). Any step that risks CT102 or the host risks all fifteen
guests. Run the non-regression checks at the bottom after anything disruptive.

# Phase 0 — Baseline and the five unknowns

## The five checks

```bash
# 1. GTT ceiling. Predicted ~31 GiB (TTM default = half of the 62GB the OS sees).
#    NOTE card1, not card0 — the iGPU on this host is card1 (226:1).
cat /sys/class/drm/card1/device/mem_info_gtt_total

# 2. Which bootloader is live — decides where kernel params go in Phase 4.
proxmox-boot-tool status
cat /proc/cmdline

# 3. What ARC is currently allowed to take.
cat /sys/module/zfs/parameters/zfs_arc_max

# 4. CPU governor (Proxmox usually lands on powersave under amd_pstate).
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# 5. Mesa and Ollama versions, and what is actually installed.
pct exec 102 -- su -s /bin/bash ollama -c 'vulkaninfo --summary' | grep -iE 'driverName|driverInfo'
pct exec 102 -- ollama --version
pct exec 102 -- ollama list
pct config 102 | grep -iE 'memory|swap'     # confirm no cgroup limit exists
```

Record the results here as they come back:

| Check | Expected / predicted | Actual | Date |
|---|---|---|---|
| `mem_info_gtt_total` | ~31 GiB | | |
| Bootloader | systemd-boot on ZFS root | | |
| `zfs_arc_max` | ~19.2 GiB | | |
| Governor | `powersave` | | |
| Mesa | 25.x, want 25.3+ | | |
| Ollama version | | | |
| CT102 memory limit | none set | | |

## The baseline benchmark

One repeatable command. Fixed prompt, fixed 200-token generation, temperature 0 so the
output is deterministic. The `model` field is echoed back **deliberately** — the source
author published a wrong attribution by measuring one model while their notes said another.

```bash
pct exec 102 -- curl -s http://127.0.0.1:11434/api/generate -d '{
  "model":"<current model>",
  "prompt":"Explain what a hypervisor does, in one paragraph.",
  "stream":false,
  "options":{"num_predict":200,"temperature":0}
}' | python3 -c 'import sys,json; d=json.load(sys.stdin); print(d["model"], round(d["eval_count"]/(d["eval_duration"]/1e9),2), "t/s")'
```

Run it **three times, keep the median.** Use this exact command for every later
measurement so the numbers are comparable.

| Stage | Model | Median t/s | Date |
|---|---|---|---|
| Baseline | | | |

## Decision gate

* **GTT returns ~31 GiB** → the ceiling is confirmed, and the "48.0 GiB" success line in
  [GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md) is indeed Ollama's
  own estimate rather than the kernel's limit. Phase 4 gains priority.
* **GTT returns ~46 GiB or more** → the arithmetic in
  [Local LLM daily driver](local-llm-daily-driver.md#the-gtt-ceiling--likely-capped-at-31-gib-here)
  is **wrong**. Correct that page before acting on anything else in it, and treat the rest
  of its predictions with more scepticism.
* **Baseline is far from ~3.2 tok/s** → the recorded figure is stale. Find out what model it
  was measured against before drawing conclusions from the comparison.

# Phase 1 — Guardrail (before pulling anything larger)

The highest-severity finding is not a speed one. On unified memory a GPU OOM **hard-locks
the whole host** — NAS, all LAN DNS, Home Assistant and ten other guests, together. Nothing
currently backstops it.

```bash
pct config 102 | grep -i memory        # record the current value first
pct set 102 -memory 40960 -swap 0      # 40GB starting point
pct reboot 102
```

**Sizing.** The limit must clear the largest model plus its KV cache, while sitting well
below the point where the host and other guests starve. With a ~31 GiB GTT ceiling, ~40GB
gives headroom without the limit ceasing to be a backstop; scale to roughly **GTT + 8GB**
if Phase 0 returned something different. The source author used 62GB against a 64GB peak,
but on a box with one other tenant rather than fourteen.

An over-tight limit shows up as OOM-kills **under load**, not at start — so confirm CT102
comes back and then actually run the benchmark before calling this done.

Two related decisions while here:

* **`OLLAMA_KEEP_ALIVE=-1`** pins models in RAM indefinitely
  ([the override](gpu-passthrough-ollama-vulkan.md#5-the-flag-ollama-needs-to-not-drop-the-igpu)).
  Recommend moving to a finite TTL now that a second model is about to exist. The source
  author unloads after 15 idle minutes.
* **ARC.** Record the ceiling; do **not** permanently shrink it — this box earns its keep
  as a NAS. The temporary-cap procedure for large loads is in
  [Local LLM daily driver](local-llm-daily-driver.md#memory-contention--the-risk-that-matters-more-than-any-speed-number).

# Phase 2 — Model shape (the main event)

Both candidates fit under even the pessimistic ~31 GiB ceiling, so **this phase does not
depend on Phase 4**:

| Candidate | Shape | Approx size | Predicted |
|---|---|---|---|
| `qwen3:30b-a3b` | ~30B MoE, ~3B active | ~19GB | ~20 t/s |
| `gpt-oss:20b` | ~20B MoE, ~3.6B active | ~13GB | ~28 t/s |

Check pool headroom first — [AIVault](/storage/aivault.md) has 164G free and a standing
reservation problem:

```bash
zfs list -o name,used,avail AIVault
pct exec 102 -- ollama pull qwen3:30b-a3b
```

Then re-run the **identical** Phase 0 benchmark against the new model.

**Expected: ~6–7× the baseline.** If that doesn't materialise, it is the most interesting
result available — stop and diagnose rather than continuing down the list, because it means
the bytes-per-token model doesn't hold here and everything downstream of it is suspect.

| Stage | Model | Median t/s | vs baseline |
|---|---|---|---|
| Phase 2 | | | |

**[sillytavern (CT120)](/containers/sillytavern.md) is a separate decision.** Its 12B dense
model is a roleplay-tuned merge; an MoE equivalent needs finding and auditioning, and the
source material's own finding is that RP finetunes and abliterated models fail in different
ways. Move the general workload first; leave CT120 until there's a candidate worth
listening to.

# Phase 3 — Cheap levers, one at a time

Re-run the same benchmark after **each** change, and record which change produced which
delta. Do not batch these.

1. **CPU governor** — the source author's top-ranked untried lever: **+36% generation,
   +120% prefill** on this exact CPU, and a different knob from GPU perf-mode (+1%, noise).
   ```bash
   cpupower frequency-set -g performance
   ```
   Instantly reversible. Note this host runs 24/7 as NAS and sole DNS, so *keeping* it is a
   power and thermal decision — measure first, decide after.
2. **Mesa** — if Phase 0 showed below 25.3, consider a newer backport (+19.8% prefill
   claimed for Valve's RADV CU-mode/LDS patches).
3. **Ollama version** — a stale bundled `llama.cpp` measured 56% slower; update if behind.
4. **Context length** — `OLLAMA_CONTEXT_LENGTH=12400` was sized against a dense model. On a
   hybrid-attention MoE (~20 KiB/token) 32K is likely affordable. Verify against real GTT
   use rather than assuming.

| Lever | Before | After | Delta | Kept? |
|---|---|---|---|---|
| Governor | | | | |
| Mesa | | | | |
| Ollama version | | | | |
| Context length | | | | |

# Phase 4 — The reboot batch (only if Phases 0–3 justify it)

Both changes want the same reboot, so do them together. Needs Dave at the console for BIOS.

## 4a. Raise the GTT ceiling

Path depends on what Phase 0's `proxmox-boot-tool status` returned. **Back up first.**

```
ttm.pages_limit=18874368 ttm.page_pool_size=18874368
```

That is 72GB (pages × 4KiB); `16777216` would be 64GB.

* **systemd-boot** (standard on ZFS root): edit `/etc/kernel/cmdline`, then
  `proxmox-boot-tool refresh`. **`/etc/default/grub` is inert — editing it does nothing.**
* **GRUB**: `/etc/default/grub`, then `update-grub`.

Do **not** use `amdgpu.gttsize` (deprecated — warns, then ignores you) or
`amdttm.pages_limit` (out-of-tree DKMS module).

## 4b. BIOS UMA 32GB → 16GB

Worth ~11% on what fits inside UMA, returns 16GB to the other fourteen guests, and lifts
OS-visible RAM to 80GB — which raises the *default* GTT ceiling to ~40 GiB independently of
4a. BIOS path is in
[GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md#the-real-performance-lever-bios-uma-frame-buffer).

## Verify from the kernel, not from the config

This is the step the source author got wrong and lost a week to. Read the values **back**:

```bash
cat /sys/class/drm/card1/device/mem_info_gtt_total    # want the raised figure
dmesg | grep -i 'VRAM:'                               # want 16384M
```

Re-check `mem_info_gtt_total` after **every `pve-kernel` upgrade** — this host's equivalent
of the source author's "re-check after every Unraid update."

Unlocks the 36–59GB model class, which is where AIVault's 164G ceiling becomes a real
blocker — see the reservation decision in [actions.md](/actions.md).

# Phase 5 — Fold results back into the bundle

Per [CLAUDE.md](/CLAUDE.md), findings must not live only in `log.md`.

* [Local LLM daily driver](local-llm-daily-driver.md) — replace the "external and
  unverified" header with measured values, keeping the source's figures alongside for
  comparison. Correct in place and dated anything the measurements refute.
* [GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md) — replace the
  ~3.2 tok/s figure with the new baseline **and the model it was measured against**.
* [ollama (CT102)](/containers/ollama.md), [sillytavern (CT120)](/containers/sillytavern.md),
  [AIVault](/storage/aivault.md) — updated model and capacity state.
* [actions.md](/actions.md) — close the two P2s (model shape, memory contention) and the P3
  (GTT ceiling) with fix details, or restate them with what was learned. Reprice the AIVault
  reservation item against actual model-store growth.
* [log.md](/log.md) — one dated entry, prose, linking every touched page.

# Rollback

No phase depends on an earlier one staying in place, so any single step can be undone
without unwinding the rest.

| Phase | Undo |
|---|---|
| 1 | `pct set 102 -memory <old value>` |
| 2 | `pct exec 102 -- ollama rm <model>` |
| 3 | `cpupower frequency-set -g <old governor>`; revert the env override |
| 4a | Restore the backed-up boot config, `proxmox-boot-tool refresh`, reboot |
| 4b | BIOS UMA back to 32GB |

# Non-regression checks

Run after any CT102 or host change. This box is the NAS and the sole DNS server:

```bash
dig @192.168.0.20 jellyfin.133gsl.ie +short      # AdGuard still resolving
curl -sS -o /dev/null -w '%{http_code}\n' https://openwebui.133gsl.ie
curl -sS -o /dev/null -w '%{http_code}\n' https://sillytavern.133gsl.ie
pct list && qm list                               # all fifteen guests running
```

Plus: a Samba share still mounts from the Mac, and
[home-assistant (VM104)](/containers/home-assistant.md) is reachable at `192.168.0.23`.

# Citations

[1] Plan derived from the review of the r/MINISFORUM N5 Pro benchmarking write-up against
this bundle, 2026-07-28 — see [Local LLM daily driver](local-llm-daily-driver.md) for the
source and its findings.

[2] Existing bundle state: [GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md),
[ollama (CT102)](/containers/ollama.md), [sillytavern (CT120)](/containers/sillytavern.md),
[AIVault](/storage/aivault.md), [actions.md](/actions.md).

[3] **No step on this page has been executed.** Written 2026-07-29 from a session with no
network route to the host; every table is empty by design, to be filled in on execution.
