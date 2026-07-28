---
type: LXC Container
title: openwebui (CT106)
description: Open WebUI chat frontend for Ollama, unprivileged LXC at 192.168.0.17.
tags: [proxmox, lxc, openwebui, ai]
timestamp: 2026-07-19T00:00:00Z
---

CT106, hostname `openwebui`, unprivileged LXC, **192.168.0.17**. Chat frontend for
[ollama (CT102)](ollama.md).

Reachable at `https://openwebui.lan` or `https://openwebui.133gsl.ie` (publicly-trusted
certificate, no per-device CA trust — see
[133gsl.ie on Cloudflare DNS](/playbooks/dns-cloudflare-133gsl-ie.md)), both proxied by
[Caddy](/playbooks/reverse-proxy-caddy.md) to `192.168.0.17:8080`.

# Citations

[1] n5-pro-homelab Claude Skill — SKILL.md (Dave's claude.ai account, last updated 2026-07-19)
