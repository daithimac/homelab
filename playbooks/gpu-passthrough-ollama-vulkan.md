---
type: Playbook
title: GPU passthrough & Ollama on Vulkan (CT102)
description: The full iGPU → Ollama-on-Vulkan chain for CT102 — device passthrough, idmap, Mesa backports, and the IGPU_ENABLE flag.
tags: [gpu, ollama, vulkan, lxc, proxmox]
timestamp: 2026-07-25T00:00:00Z
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
Environment="OLLAMA_CONTEXT_LENGTH=12400"
Environment="OLLAMA_IGPU_ENABLE=1"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_KEEP_ALIVE=-1"
EOF
systemctl daemon-reload
systemctl restart ollama
```

Verify the flag actually reached the process (the check that never lies —
`systemctl edit` in nano has silently failed via a stray lock file before, so confirm the
live value):
```bash
systemctl show ollama -p Environment | tr ' ' '\n' | grep -i igpu    # want OLLAMA_IGPU_ENABLE=1
journalctl -u ollama --no-pager --since "30 sec ago" | grep -iE 'vulkan|vram|dropping|inference compute'
```
Win = `dropping` line GONE, and
`inference compute ... library=Vulkan ... type=iGPU total="48.0 GiB"`. (It reports ~48GB
because the iGPU can address system RAM as GTT, so capacity is NOT capped at the 2GB
dedicated VRAM — but see the speed note below.)

# Tuning notes

* `OLLAMA_FLASH_ATTENTION=1` + `OLLAMA_KV_CACHE_TYPE=q8_0` only matter once the GPU
  backend is live (they did nothing on CPU). q8_0 halves KV cache, nearly lossless — worth
  it with the 12400 context.
* `OLLAMA_KEEP_ALIVE=-1` pins models in memory (shows as a giant duration in logs —
  normal). With q8 cache + 12400 ctx on a shared-DDR5 iGPU, watch for memory pressure —
  it's the first suspect if loads crawl or OOM.
* Discovery may set a huge `default_num_ctx` (e.g. 262144) because it sees ~48GB. The
  explicit `OLLAMA_CONTEXT_LENGTH=12400` overrides per-model; keep it set.

# The real performance lever: BIOS UMA frame buffer

Measured generation was ~3.2 tok/s on shared memory — iGPU inference is
**memory-bandwidth bound** on DDR5, so this is inherent, not a misconfig. More CPU cores
don't help (bandwidth, not compute, is the limit). The one hardware lever is dedicated
VRAM in BIOS:

1. Shut down guests cleanly, reboot host, enter BIOS (`Del` or `F2`).
2. Path is firmware-dependent: `Advanced → AMD CBS → NBIO Common Options → GFX
   Configuration` (or `UMA Frame Buffer Size`), set UMA mode to
   `UMA_SPECIFIED`/`Manual`.
3. **Currently 32GB** (`dmesg | grep -i 'VRAM:'` → `VRAM: 32768M`, confirmed live
   2026-07-25 — this doc previously said "currently 2GB," which was stale; someone
   applied a BIOS change at some point without updating this line). This is *above* the
   documented sweet spot: **24GB** covers a q4 30B model (~18GB) + KV headroom while
   leaving 72GB for host + guests out of 96GB physical; 16GB if only running ≤13B models;
   32GB was flagged as "defensible but marginal past 24" even before this was checked —
   right now it's quietly costing every other guest 8GB of headroom versus the 24GB
   target, for no measured inference benefit over 24GB. See [actions.md](/actions.md) for
   the decision on whether to dial it back.
4. Save (`F10`), reboot. Verify on host: `dmesg | grep -i 'VRAM:'` reads back your target
   (e.g. `24576M` for 24GB).

Dedicated UMA gives cleaner allocation for hot weights than GTT spill, but it's a
**modest** gain, not the CPU→GPU night-and-day. A 30B model will still be slow; match
model size to the hardware (7–14B q4 is the pleasant range).

# Citations

[1] n5-pro-homelab Claude Skill — references/gpu-ollama.md (Dave's claude.ai account, last updated 2026-07-19)
