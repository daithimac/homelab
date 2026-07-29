---
type: LXC Container
title: ollama (CT102)
description: Ollama LLM inference on Vulkan/iGPU, at 192.168.0.13.
tags: [proxmox, lxc, ollama, gpu, ai]
timestamp: 2026-07-29T00:00:00Z
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
Symptoms that send you to that playbook: `library=cpu`, `total_vram="0 B"`, `dropping
integrated GPU`, or slow inference.

# Storage

Model store is `AIVault/ollama-models` on the [AIVault](/storage/aivault.md) pool,
bind-mounted into this container at `/mnt/models` (`OLLAMA_MODELS=/mnt/models`).

## Model inventory (2026-07-29)

First actual inventory of the store — previously only one model was documented anywhere in
the bundle, leaving ~80G unexplained. It is **not** unexplained: seven models, ~94 GB
nominal (89.7G on disk), no orphaned blobs, nothing to reclaim.

| Model | Size | Last modified |
|---|---|---|
| `DavidAU/Qwen3.6-27B-Fable-Fusion-711-Uncensored-Heretic-NM-DAU-NEO-MAX-MTP-GGUF:latest` | 18 GB | 2026-07-28 |
| `ReadyArt/Melody1437-26B-A4B-v2.0-GGUF:latest` | 17 GB | ~2026-07-15 |
| `mradermacher/gemma-4-26B-A4B-it-ultra-uncensored-heretic-i1-GGUF:Q4_K_M` | 16 GB | ~2026-07-08 |
| `bartowski/cognitivecomputations_Dolphin-Mistral-24B-Venice-Edition-GGUF:Q8_0` | 25 GB | ~2026-07-08 |
| `Ttimofeyka/MistralRP-Noromaid-NSFW-Mistral-7B-GGUF:Q8_0` | 7.7 GB | 2026-07-19 |
| `mradermacher/Moonlit-Mirage-12B-i1-GGUF:latest` | 7.5 GB | ~2026-07-08 |
| `wkplhc/Qwen3.5-4B-NSFW-ARA-Heretic-Literotica-i1-GGUF:latest` | 2.7 GB | 2026-07-19 |

Worth noting rather than acting on: four of the seven (76 GB of the 94) are well above the
7–14B q4 range the playbook calls "the pleasant range" on this hardware, and the 24B is a
Q8 rather than a q4. They are not dead weight — but they will be slow, and with AIVault no
longer capped at 164G (see [AIVault](/storage/aivault.md)) there is now room to keep them
without pressure either way.

# Inference performance

Measured 2026-07-29: **~10.9 tok/s** for `Moonlit-Mirage-12B` (i1/q4) and **~23.8 tok/s**
for `Qwen3.5-4B` (i1/q4), both 100% GPU at 16384 context. Full method, per-run figures and
reproduction instructions are in the playbook's
[Baseline section](/playbooks/gpu-passthrough-ollama-vulkan.md#baseline).

**The bundle's long-standing `~3.2 tok/s` figure was wrong**, not merely unqualified — the
real 12B number is ~3.4× higher. It has been retired from the playbook; don't cite it.

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
