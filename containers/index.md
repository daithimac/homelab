# Containers & VMs

Fifteen guests on the [N5 Pro host](/host/n5-pro.md), subnet `192.168.0.x/24`. All use
static IPs in `.11-.27`, DNS pointed at [AdGuard](/containers/adguard.md)
(`192.168.0.20`), and IPv6 off — see [IP addressing](/network/ip-addressing.md) for the
policy history and how each was verified.

* [nas (CT100)](nas.md) - Samba / NAS, unprivileged LXC, 192.168.0.11.
* [jellyfin (CT101)](jellyfin.md) - Media server with hardware transcode via the iGPU, unprivileged LXC, 192.168.0.12.
* [ollama (CT102)](ollama.md) - Ollama LLM inference on Vulkan/iGPU, unprivileged LXC, 192.168.0.13.
* [docker-stack (VM103)](docker-stack.md) - Ubuntu VM running Postgres, Qdrant, n8n in Docker, 192.168.0.14.
* [home-assistant (VM104)](home-assistant.md) - Home Assistant OS VM, 192.168.0.23 (static, fixed 2026-07-25 — was DHCP-leased at .213).
* [tailscale (CT105)](tailscale.md) - Tailscale subnet router + exit node, unprivileged LXC, 192.168.0.16.
* [openwebui (CT106)](openwebui.md) - Open WebUI chat frontend for Ollama, unprivileged LXC, 192.168.0.17.
* [opencode (CT107)](opencode.md) - OpenCode agentic Proxmox manager, LXC, 192.168.0.18.
* [sabnzbd (CT108)](sabnzbd.md) - SABnzbd Usenet downloader onto MediaTank, LXC, 192.168.0.19.
* [adguard (CT109)](adguard.md) - AdGuard Home, tailnet-wide DNS + ad blocking, unprivileged LXC, 192.168.0.20.
* [grafana (CT110)](grafana.md) - Grafana dashboards, unprivileged LXC, 192.168.0.21.
* [audiobooks (CT111)](audiobooks.md) - EPUB/PDF to audiobook converter with web UI, unprivileged LXC, 192.168.0.24.
* [audiobookshelf (CT112)](audiobookshelf.md) - Audiobookshelf server, plays the library CT111 writes, unprivileged LXC, 192.168.0.26.
* [code-server (CT113)](code-server.md) - VS Code in the browser, unprivileged LXC, 192.168.0.27. Served at `coder.lan`, not `code-server.lan`.
* [sillytavern (CT120)](sillytavern.md) - SillyTavern chat frontend for ollama, unprivileged LXC, 192.168.0.22.
