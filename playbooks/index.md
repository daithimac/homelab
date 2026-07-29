# Playbooks

Step-by-step, hard-won procedures for this specific host. Check
[Recurring gotchas](troubleshooting-gotchas.md) FIRST when something breaks — it's the
fast path to a known fix.

* [Recurring gotchas](troubleshooting-gotchas.md) - the checklist to run through before first principles.
* [GPU passthrough & Ollama on Vulkan](gpu-passthrough-ollama-vulkan.md) - the full iGPU → Ollama chain for CT102.
* [Local LLM daily driver](local-llm-daily-driver.md) - getting usable tokens/sec out of the 890M once passthrough works: model shape, the GTT ceiling, the UMA trade, memory contention.
* [AI optimisation runbook](ai-optimisation-runbook.md) - the executable procedure for the above: baseline, guardrail, model swap, cheap levers, reboot batch. Not yet run.
* [Tailscale subnet router & exit node](tailscale-subnet-router-exit-node.md) - setup for CT105.
* [ZFS pools & bind-mount permissions](zfs-bind-mount-permissions.md) - the UID-mapping dance for unprivileged LXCs.
* [Reverse proxy via Caddy](reverse-proxy-caddy.md) - how `.lan` domains get real HTTPS with no port, on docker-stack (VM103).
* [133gsl.ie on Cloudflare DNS](dns-cloudflare-133gsl-ie.md) - the public domain, split-horizon DNS, and publicly-trusted wildcard certs with no open ports.
* [EPUB/PDF to audiobook](epub-to-audiobook.md) - converting ebooks to MP3 with Kokoro TTS and CT111.
