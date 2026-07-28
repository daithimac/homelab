---
type: LXC Container
title: ollama (CT102)
description: Ollama LLM inference on Vulkan/iGPU, at 192.168.0.13.
tags: [proxmox, lxc, ollama, gpu, ai]
timestamp: 2026-07-19T00:00:00Z
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

# Storage

Model store is `AIVault/ollama-models` on the [AIVault](/storage/aivault.md) pool,
bind-mounted into this container at `/mnt/models` (`OLLAMA_MODELS=/mnt/models`).

# Frontend

[openwebui (CT106)](openwebui.md) is the chat frontend for this container.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md, references/gpu-ollama.md (Dave's claude.ai account, last updated 2026-07-19)
