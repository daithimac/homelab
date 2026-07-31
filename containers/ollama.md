---
type: LXC Container
title: ollama (CT102)
description: Ollama LLM inference on Vulkan/iGPU, at 192.168.0.13.
tags: [proxmox, lxc, ollama, gpu, ai]
timestamp: 2026-07-30T00:00:00Z
---

CT102, hostname `ollama`, **192.168.0.13**. Runs Ollama for LLM inference against the
Radeon 890M iGPU via the Vulkan backend (not ROCm — gfx1150 support under ROCm is flaky).

The API is on `192.168.0.13:11434`, also proxied as `https://ollama.lan` /
`https://ollama.133gsl.ie` ([133gsl.ie on Cloudflare DNS](/playbooks/dns-cloudflare-133gsl-ie.md)).
Note the in-fleet clients — [openwebui](openwebui.md) and [sillytavern](sillytavern.md) —
point at the **raw `IP:11434`**, not the proxied name, and there's no reason to change that:
routing local service-to-service calls through Caddy would only add a hop and a TLS
handshake.

# Privilege mode — confirm before assuming

As of the GPU-passthrough work this container remains **unprivileged**, with an idmap
carve-out mapping the host's render group through (Dave weighed converting to privileged
but took the idmap path instead). Confirm before assuming:

```bash
grep unprivileged /etc/pve/lxc/102.conf
```

# GPU passthrough & inference

Full device passthrough, idmap, Mesa backports, and the `OLLAMA_IGPU_ENABLE=1` chain are
documented in [GPU passthrough & Ollama on Vulkan](/playbooks/gpu-passthrough-ollama-vulkan.md).
Symptoms that send you to that playbook: `library=cpu`, `total_vram="0 B"`, or `dropping
integrated GPU`.

**Slow inference is a different problem and now has its own page** —
[Local LLM daily driver](/playbooks/local-llm-daily-driver.md). Headline: the ~3.2 tok/s
recorded here matches what the bandwidth model predicts for a **dense** ~30B q4 model, so
the chain is healthy and the *model shape* is the bottleneck; a ~30B **MoE** with ~3B
active should run roughly 7× faster. That page also covers the GTT ceiling (possibly
capping this container at ~31 GiB regardless of Ollama's reported 48 GiB), the CPU
governor, and a memory-contention risk that can hard-lock the host.

**No memory limit is set on this container.** On unified memory a GPU OOM can lock the
whole box rather than just this guest — worth a cgroup limit before experimenting with
larger models. See the memory-contention section of that page.

The iGPU memory pool is **104 GiB — 32 GB BIOS UMA + 72 GiB GTT** (`ttm.pages_limit=18874368`
on the host kernel command line since 2026-07-29; it was 63.2 GiB at the default GTT before
that). Ollama's `inference compute` journal line reports `total="104.0 GiB"`. The raise is
capacity, not speed — the full 7-model benchmark after it measured flat. Details and
verification in the playbook's
[UMA and GTT section](/playbooks/gpu-passthrough-ollama-vulkan.md#uma-and-gtt--the-memory-pool-and-how-its-split).

# Storage

Model store is `AIVault/ollama-models` on the [AIVault](/storage/aivault.md) pool,
bind-mounted into this container at `/mnt/models` (`OLLAMA_MODELS=/mnt/models`).

## Model inventory (2026-07-29)

First actual inventory of the store — previously only one model was documented anywhere in
the bundle, leaving ~80G unexplained. It is **not** unexplained: seven models, ~83 GB
nominal (80G on disk, `zfs list -o used AIVault/ollama-models` = 79.8G / `du -sh` = 80G,
2026-07-29), no orphaned blobs, nothing to reclaim. (The store briefly held an eighth model —
the 24B `Q8_0` — for a Q4-vs-Q8 A/B; it was removed 2026-07-29 once the owner decided, see
below.)

Sorted by measured speed, not size — because on this hardware the two are only loosely
related (see below). **The tok/s column is the pre-GTT-raise Baseline figure for the six
other shared-harness models** (the post-raise `gtt72` re-run moved every one of them by
≤2.2%, see the playbook's [gtt72 table](/playbooks/gpu-passthrough-ollama-vulkan.md#uma-and-gtt--the-memory-pool-and-how-its-split))
**— except the Q4_K_M row, which is post-`gtt72` by construction**, pulled and benched
after the raise specifically for the Q8 A/B (below).

| Model | Type | Size | **tok/s** | Verdict |
|---|---|---|---|---|
| `mradermacher/gemma-4-26B-A4B-it-ultra-uncensored-heretic-i1-GGUF:Q4_K_M` | **MoE ~4B** | 16 GB | **24.82** | **Fastest model here** |
| `wkplhc/Qwen3.5-4B-NSFW-ARA-Heretic-Literotica-i1-GGUF:latest` | dense | 2.7 GB | 23.54 | Fast, small |
| `ReadyArt/Melody1437-26B-A4B-v2.0-GGUF:latest` | **MoE ~4B** | 17 GB | **19.71** | Strong |
| `mradermacher/Moonlit-Mirage-12B-i1-GGUF:latest` | dense | 7.5 GB | 10.88 | Usable |
| `Ttimofeyka/MistralRP-Noromaid-NSFW-Mistral-7B-GGUF:Q8_0` | dense | 7.7 GB | 10.43 | Usable |
| `bartowski/cognitivecomputations_Dolphin-Mistral-24B-Venice-Edition-GGUF:Q4_K_M` | dense | 14 GB | **5.63** | **Kept over the Q8 (removed)** |
| `DavidAU/Qwen3.6-27B-Fable-Fusion-711-...-MTP-GGUF:latest` | dense | 18 GB | **4.28** | Slow for its size |

**The headline: the fastest model in the store is also one of the largest.** The 16 GB
gemma MoE beats the 7.5 GB dense 12B by 2.3×, and beats the 18 GB dense 27B — nearly the
same file size — by **5.8×**. Only ~4B of its 26B parameters are active per token, and
bytes-read-per-token is what sets speed on a bandwidth-bound iGPU.

Practical reading: **check the name for an `A<n>B` suffix before the file size.** `26B-A4B`
means 26B total, 4B active, and it will run like a 4B. Full reasoning and the prediction
rule for dense models are in the playbook's
[bytes-per-token section](/playbooks/gpu-passthrough-ollama-vulkan.md#bytes-read-per-token-is-the-lever--which-means-moe-not-size).

One thing acted on, tracked in [actions.md](/actions.md): the 24B was available only as
**Q8**, and the dense rule (`~80 ÷ size-in-GB`) predicted a Q4_K_M at ~13-14 GB would land
around **5.7-6.2 tok/s** against the Q8's 3.22 tok/s (post-`gtt72` figure for the
since-removed model). **Measured: the Q4 ran 1.75× the Q8** (5.63 vs 3.22, predicted
5.7-6.2) — 3 runs, pulled and benched 2026-07-29, 14 GB on disk, just under the predicted
band's low end. 14 GB × 5.63 = **79**, in line with the rule's ~80 constant: the dense
rule's track record held up here to within a few percent. **Decided: the owner kept the
Q4_K_M and removed the Q8_0, 2026-07-29** — the quality difference at this size was judged
not worth the Q8's ~1.75× slower throughput. `ollama rm` was run against the Q8_0 tag; the
Q4_K_M row above is what remains in the store.

That quant-level decision is now settled; the other item stays open: whether the whole
24B/27B dense family is worth keeping at all, since the two slowest dense models here — the
27B and the 24B `Q4_K_M` — are doing nothing the MoEs don't do faster. Storage is no longer
a reason to prune — AIVault is no longer capped at 164G (see
[AIVault](/storage/aivault.md)) — but speed is, and that call belongs to the owner.

# Inference performance

Six of the seven models present were benchmarked three-run on 2026-07-29 (Baseline +
`gtt72`, all 100% GPU at 16384 context) via `/root/bench-ollama.sh` — see the table above
and the playbook's [Baseline section](/playbooks/gpu-passthrough-ollama-vulkan.md#baseline)
for method and reproduction. The 24B Q4_K_M was pulled and benched separately, after both of
those (label `q4-vs-q8`, post-`gtt72`), replacing the 24B Q8_0 that was in that original run
and has since been removed (see above). Spread across the seven rows now tabulated is
**4.28 → 24.82 tok/s (5.8×)** on identical hardware and identical config — the historical
low of **3.26 tok/s** belonged to the removed Q8_0 and no longer applies to any model in the
store.

**For dense models throughput is predictable: roughly `80 ÷ size in GB` tok/s**, accurate
to a few percent above ~7 GB. **For MoE models that rule is wrong by ~5×** and must not be
applied — only active experts are read per token. An earlier revision of these notes stated
the rule without the dense qualifier; every model measured at that point happened to be
dense, which is exactly why the error looked validated.

**On the old `~3.2 tok/s` figure:** it cannot be attributed, and two attempts to settle it
were both wrong — first "wrong by ~3.4×" (compared against the 12B), then "correct all
along" (compared against the 24B Q8 at 3.26). Both inferred a model from one matching
number. All it genuinely constrains is **~25 GB of dense weights**, satisfied by a 24B at
Q8 or a ~45B at Q4, with nothing on record to distinguish them.

## Concurrency bounds

`OLLAMA_MAX_LOADED_MODELS=1` as of 2026-07-29 (was `2`). Combined with
`OLLAMA_KEEP_ALIVE=-1`, the old value let two models sit pinned in memory indefinitely —
observed live at 8.8 GB + 3.2 GB = 12 GB held with both idle. At `1`, loading a second
model evicts the first (verified), and single-stream throughput is unchanged.

`OLLAMA_NUM_PARALLEL=1` with `OLLAMA_MAX_QUEUE=32` — one request slot, so openwebui and
sillytavern queue behind each other rather than splitting KV cache. Deliberately left at 1.

Check these against the **live process**, never the file on disk:

```bash
pct exec 102 -- systemctl show ollama -p Environment | tr ' ' '\n'
```

# Frontend

[openwebui (CT106)](openwebui.md) is the chat frontend for this container.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/gpu-ollama.md (Dave's claude.ai account, last updated 2026-07-19)
[2] direct host review 2026-07-29 — `ollama list`, `ollama ps`, `ollama run --verbose` benchmark runs, `systemctl show ollama -p Environment` (model inventory, inference performance, concurrency bounds)
