---
type: Reference
title: Open Actions
description: Prioritised list of known issues and follow-ups surfaced during documentation review, not yet actioned.
tags: [todo, actions, homelab]
timestamp: 2026-07-25T00:00:00Z
---

# Open Actions

Standing list of confirmed issues and open questions that need a deliberate fix, kept
separate from the reference docs so nothing gets lost. Update this when an item is closed
or a new one is found — don't let findings just live in [log.md](log.md) history.

## Resolved

* ~~**SSH from Dave's Mac to the host was broken, blocking all live-state verification.**~~
  **Fixed 2026-07-26.** `ssh root@192.168.0.10` returned `Permission denied
  (publickey,password)`. Diagnosed with `ssh -v` first: the key *was* being offered and
  the server rejected it, so it was never a client, agent or `~/.ssh/config` problem.
  Confirmed host-side via the Proxmox web console — `/root/.ssh/authorized_keys` is a
  symlink to the cluster-managed `/etc/pve/priv/authorized_keys`, which held three keys,
  none of them the Mac's current ones. `sshd -T` showed nothing wrong
  (`permitrootlogin yes`, `pubkeyauthentication yes`), ruling out config. Backed up to
  `/root/authorized_keys.bak-20260726`, appended the Mac's `id_ed25519` public key
  (deliberately the default identity, so plain `ssh root@192.168.0.10` works with no
  config), and verified the appended line by **fingerprint** rather than eyeballing the
  base64. Confirmed working end-to-end: `pveversion` over SSH, then `pct exec 111` for the
  two CT111 checks that had been blocked all session. `pve-backup` was left unused — it's
  a keypair with no config referencing it. See
  [n5-pro](/host/n5-pro.md#ssh-access). Left open above: the two stale keys found alongside.

  **This regressed and was re-fixed 2026-07-28.** Checked live at the start of the DNS work:
  `~/.ssh/` on the Mac contained **no `.pub` files at all**, `ssh-add -l` reported no
  identities, and both `root@192.168.0.10` and `dave@192.168.0.14` returned `Permission
  denied (publickey,password)`. So the `id_ed25519` installed on 2026-07-26 no longer exists
  on the Mac — the keypair was lost or the machine was rebuilt between sessions, and the
  entry above read as current while being false. Dave generated a **new** `id_ed25519` and
  `ssh-copy-id`'d it to both `root@192.168.0.10` and `dave@192.168.0.14` (the VM, which had
  never had key auth before). Verified from the Mac with `ssh -o BatchMode=yes`. Two
  consequences: the P3 item below about stale keys now describes fingerprints that **no
  longer match anything on the Mac either**, since the Mac's own key changed again; and the
  host's `authorized_keys` has accumulated another entry. Worth a single reconciliation pass
  over `/etc/pve/priv/authorized_keys` rather than another append next time.

* ~~**Edge TTS was documented as the Irish route without anything having been synthesised
  through it.**~~ **Verified 2026-07-26, same day it was raised.** Dave chose Edge after
  it turned out neither local backend has any Irish voice (both checked live — Kokoro's
  60-odd voices contain no `ie`, the UI's Piper tab offers 38 locales whose only English
  entries are `en_GB`/`en_US`). The open question was whether
  [CT111](/containers/audiobooks.md) could actually reach Microsoft, since the UI's voice
  dropdown is a static constant that answers without a network call. Settled by rendering
  real audio: a one-chapter EPUB through `en-IE-ConnorNeural` produced
  `/srv/media/Audiobooks/edge-ie-smoketest/0001_Smoke_Test.mp3`, 183 characters in ~1.4s —
  so `edge_tts` is in the venv and egress works. The `pct exec` checks originally proposed
  here were never run (SSH to the host refuses both keys on Dave's Mac); the test went
  through the UI's own Gradio API instead. Found in the same log: a full 30-chapter book
  had **already** converted successfully through Edge earlier that morning at 8 workers
  concurrently, so the caveat was stale the moment it was written. See
  [Edge is verified working](/playbooks/epub-to-audiobook.md#edge-is-verified-working-on-this-box).
  Leftover test artifact to bin whenever: `/srv/media/Audiobooks/edge-ie-smoketest/`.

* ~~**docker-stack's 200G AIVault disk was attached but completely unused.**~~ **Migrated
  2026-07-25.** Dave chose to wire it in. Formatted ext4, mounted at `/mnt/aivault`,
  `rsync -aHAX`'d `/data/postgres` and `/data/qdrant` onto it (verified byte-identical
  with `diff -rq` before touching originals), then bind-mounted the new disk back onto the
  original paths via `/etc/fstab` — `docker-compose.yml` needed zero changes. Containers
  stopped for consistency during the copy, restarted after; verified Postgres recognized
  the existing database and reached "ready to accept connections", Qdrant came up clean,
  n8n (Postgres-dependent) answered `200` throughout. See
  [AIVault](/storage/aivault.md#postgresqdrant-migrated-here-2026-07-25).

* ~~**`MediaTank/Media` and `MediaTank/media` are two different directories differing only
  by case.**~~ **Investigated and decided 2026-07-25 — left as-is.** Checked
  [nas (CT100)](/containers/nas.md)'s `smb.conf` first: only one share exists,
  `[media]` → `/srv/media` (the active lowercase one) — `Media` isn't exposed over Samba
  at all right now, so this isn't a live collision today. `Media`'s content (dated
  2026-07-04, untouched since, named like typical downloaded releases) reads as an
  orphaned pre-reorganization download landing spot, not a Mac backup as first guessed.
  Dave chose to leave both as-is rather than rename anything. See
  [MediaTank](/storage/mediatank.md#media-vs-media--two-different-directories-case-is-the-only-difference-found-2026-07-25).

* ~~**Stray manual snapshot** `DataPool/subvol-107-disk-0@test-snap`~~ **Deleted
  2026-07-25.** `zfs destroy DataPool/subvol-107-disk-0@test-snap`. Confirmed nothing
  depended on it before removing. See
  [DataPool](/storage/datapool.md#stray-snapshot-found-2026-07-25).

* ~~**Leftover TrueNAS system datasets on MediaTank**~~ **Removed 2026-07-25.**
  `zfs destroy -r MediaTank/.system` — confirmed all six were `mountpoint: legacy`
  (never auto-mounted, nothing reading them) before removing. Pool root now only holds
  `Media`, `media`, `backups`. See
  [MediaTank](/storage/mediatank.md#leftover-truenas-system-datasets--removed-2026-07-25).

* ~~**Dead leftover NIC/bridge from the abandoned NAT-over-TB experiment.**~~ **Removed
  2026-07-25.** `pct set 100 -delete net1` plus the `vmbr1` stanza dropped from the host's
  `/etc/network/interfaces` (backed up first), applied live via `ifreload -a`. Verified:
  `vmbr1` no longer exists, no `eth1` inside [nas](/containers/nas.md), main connectivity
  and DNS unaffected. See [Thunderbolt link](/network/thunderbolt-link.md).

* ~~**Host RAM figure is disputed.**~~ **Explained 2026-07-25, not an error.** Docs said
  96GB, Proxmox's summary showed ~62GB. Both are correct: `dmidecode -t memory` confirms
  96GB physically installed (2×48GB DIMMs); `free -h` shows 62GB because the iGPU's BIOS
  UMA frame buffer reserves 32GB for itself before the OS ever sees it
  (`dmesg`: `VRAM: 32768M`). Not a capacity-planning error — but see the P2 item below,
  since 32GB is more than the documented 24GB sweet spot for that reservation.

* ~~**`https://jellyfin.lan` (and every other `.lan` service) didn't work — no reverse
  proxy, so HTTPS on the default port had nothing to connect to.**~~ **Fixed 2026-07-25.**
  Root cause turned out to be more than a missing feature: a fully-configured `caddy:2`
  container (`stack-caddy-1`) already existed on
  [docker-stack (VM103)](/containers/docker-stack.md), with automatic internal HTTPS
  already solved for nearly every `.lan` domain — just never connected to DNS, and
  completely undocumented. Confirmed it actually worked (`curl` through it directly
  returned correct responses) before deciding whether to build a redundant new proxy;
  Dave chose to wire up the existing one instead. Added a missing `sillytavern.lan` block,
  reloaded Caddy live (`caddy reload`, no downtime), then repointed 9 AdGuard rewrites at
  docker-stack (`192.168.0.14`) and added a `qdrant.docker.lan` entry Caddy already
  supported but AdGuard never had. Hit and fixed two real bugs along the way: an
  AdGuardHome YAML indentation mismatch (crash-looped the service — see the playbook for
  the precise column measurements) and a SABnzbd `host_whitelist` 403 (fixed by adding
  `sabnzbd.lan` to `sabnzbd.ini` and restarting). Verified all 8 newly-proxied domains
  end-to-end (real DNS resolution → Caddy → correct backend, `200`/`302` as expected). See
  [Reverse proxy via Caddy](/playbooks/reverse-proxy-caddy.md) for the full domain table
  and setup detail, and the new P4 item above about trusting Caddy's CA.

* ~~**IPv6 was live on 9 of 12 guests despite documented "IPv6 off" policy.**~~ **Fixed
  2026-07-25.** Root cause: only [openwebui (CT106)](/containers/openwebui.md) had ever
  had it explicitly disabled (`/etc/sysctl.d/99-disable-ipv6.conf`, set by the Proxmox
  community-scripts installer at creation) — nothing else in the fleet had the same
  treatment. Raised as a decision rather than assumed (fix vs. drop the policy); Dave chose
  to fix it fleet-wide. Replicated the same sysctl file
  (`net.ipv6.conf.{all,default,lo}.disable_ipv6 = 1`) on the other 8 LXCs (nas, jellyfin,
  ollama, opencode, sabnzbd, adguard, grafana, sillytavern) and on docker-stack (VM103, via
  `qm guest exec`). Applied **live** via `sysctl -p` — unlike the IP/DNS fixes, no reboot
  was needed; global IPv6 addresses drop immediately. Verified `ip -6 addr show scope
  global` returns nothing on all 9, and IPv4 connectivity on docker-stack confirmed
  unaffected. [tailscale (CT105)](/containers/tailscale.md) already had no global IPv6 for
  unrelated reasons; [home-assistant (VM104)](/containers/home-assistant.md) was fixed
  earlier the same day. All 12 guests now match policy. See
  [IP addressing](/network/ip-addressing.md#ipv6--off-fleet-wide-as-of-2026-07-25).

* ~~**7 more LXCs (nas, jellyfin, ollama, openwebui, opencode, sabnzbd, grafana) also
  pointed DNS at the router, not AdGuard.**~~ **Fixed 2026-07-25**, same audit pass that
  found sillytavern's DHCP/DNS drift. This one wasn't simple drift, though — the *original*
  documented policy in [DNS via AdGuard](/network/dns-adguard.md) said guests should
  inherit DNS from the host (confirmed: the Proxmox host's own `/etc/resolv.conf` is also
  `192.168.0.1`), and AdGuard was meant for tailnet clients only. That directly
  contradicted the docker-stack/home-assistant/sillytavern fixes already made that day.
  Asked Dave to settle it: **guests should use AdGuard**, so it was rolled out fleet-wide
  instead of reverting the other three. Applied via `pct set <id> -nameserver 192.168.0.20`
  for all 7 — config saved immediately but didn't take effect on the running containers
  until each was rebooted (`pct reboot`, one at a time, done with explicit go-ahead since
  it briefly interrupts active services — NAS/Samba, Jellyfin, Ollama, SABnzbd downloads).
  Verified post-reboot: all 7 show `nameserver 192.168.0.20` and all came back up
  `running`. [adguard (CT109)](/containers/adguard.md) (self-reference) and
  [tailscale (CT105)](/containers/tailscale.md) (Tailscale's own MagicDNS) intentionally
  excluded. See [DNS via AdGuard](/network/dns-adguard.md#guest-dns-policy--changed-2026-07-25).

* ~~**sillytavern (CT120) was on DHCP, not static, and pointed DNS at the router.**~~
  **Fixed 2026-07-25.** Converted at the Proxmox level (not inside the guest — LXC
  networking is host-managed): `pct set 120 -net0
  name=eth0,bridge=vmbr0,hwaddr=BC:24:11:4A:51:E9,ip=192.168.0.22/24,gw=192.168.0.1,type=veth
  -nameserver 192.168.0.20`, then `pct reboot 120`. Verified after reboot: static `.22/24`
  in `ip -4 addr show eth0` (no DHCP lease markers), `/etc/resolv.conf` shows AdGuard
  (`192.168.0.20`), SillyTavern's web UI answers `200` on port 8000. See
  [sillytavern (CT120)](/containers/sillytavern.md).

* ~~**grafana (CT110) and sillytavern (CT120) docs are stubs.**~~ **Filled in 2026-07-25**
  via live `pct exec` shell review of both containers. Grafana: Grafana 13.1.0 + a
  co-located Prometheus scraping the Proxmox host through `pve-exporter`, no dashboards
  built yet. SillyTavern: native Node.js (not Docker), talks directly to
  [ollama (CT102)](/containers/ollama.md) at `192.168.0.13:11434`. Surfaced the DHCP/DNS
  issue above in the process. The CT120 numbering-jump reason remains unconfirmed (no
  leftover volume evidence either way). See [grafana](/containers/grafana.md) and
  [sillytavern](/containers/sillytavern.md).

* ~~**home-assistant (VM104) was on DHCP, not static, and bypassed AdGuard for DNS.**~~
  **Fixed 2026-07-25.** Moved to static `192.168.0.23`, gateway `192.168.0.1`, DNS
  `192.168.0.20` (AdGuard), IPv6 disabled — via `ha network update enp6s18` in the HAOS
  CLI. Verified: `network info` shows `method: static`; `.23` reachable, old `.213` lease
  gone. See [home-assistant (VM104)](/containers/home-assistant.md) and
  [IP addressing](/network/ip-addressing.md).

* ~~**docker-stack (VM103)'s IP (192.168.0.14) — static or DHCP unconfirmed.**~~
  **Checked 2026-07-25 — confirmed static**, no fix needed. Read the guest's own
  `/etc/netplan/50-cloud-init.yaml` via `qm guest exec 103` (no SSH needed): address is
  hardcoded, no `dhcp4`. It just happens to sit in the historical `.12/.14/.16` collision
  zone by coincidence, not an active conflict. See
  [docker-stack (VM103)](/containers/docker-stack.md).

* ~~**docker-stack (VM103) pointed DNS at the router (192.168.0.1), not AdGuard.**~~
  **Fixed 2026-07-25.** Patched `/etc/netplan/50-cloud-init.yaml`'s nameserver to
  `192.168.0.20` via `qm guest exec` (no SSH needed), `netplan apply`d. Verified with
  `resolvectl status enp6s18`. While verifying, surfaced the much bigger AdGuard rewrite
  bug below — now also fixed. See [docker-stack (VM103)](/containers/docker-stack.md).

* ~~**AdGuard's rewrite table had 7 of 12 domains miscopied to `192.168.0.14`
  (docker-stack).**~~ **Fixed 2026-07-25.** `jellyfin.lan`, `openwebui.lan`,
  `opencode.lan`, `sabnzbd.lan`, `adguard.lan`, `proxmox.lan`, and `grafana.lan` were all
  wrongly answering `192.168.0.14` instead of their own statics (`.12`, `.17`, `.18`,
  `.19`, `.20`, `192.168.0.10`, `.21`) — a copy-paste error while adding entries.
  `n8n.docker.lan` legitimately answers `.14`, left untouched. Backed up
  `AdGuardHome.yaml` first (`.bak-20260725`), patched each domain's `answer` field via
  `pct exec 109` with a `sed` anchored per-domain block, `systemctl restart AdGuardHome`.
  Verified live with `dig @192.168.0.20` against all 12 domains post-restart — all
  correct. Full table in [DNS via AdGuard](/network/dns-adguard.md).

## P1 — confirmed, act soon

* **`/opt/stack/docker-compose.yml` on docker-stack holds plaintext credentials in a
  world-readable file.** Found 2026-07-28 while adding the `env_file` for the Cloudflare
  token. The file is mode `664` — readable by every account on
  [VM103](/containers/docker-stack.md) — and carries `POSTGRES_PASSWORD` on the `postgres`
  service and `N8N_SMTP_PASS` on `n8n`. The second is the one that matters: it's a **Google
  app password for Dave's Gmail**, which bypasses 2FA and grants SMTP send as him, sitting
  next to `N8N_SMTP_USER`/`N8N_SMTP_SENDER` so it's immediately usable by anyone who reads
  the file. Rotate the app password at
  [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords) — only Dave
  can do that — then move both secrets into a `600` env file referenced via `env_file:`,
  exactly as `/opt/stack/caddy.env` now does for the Cloudflare token, and tighten the
  compose file to `640`. Note that rotating `POSTGRES_PASSWORD` also means an `ALTER USER`
  inside the running container plus updating consumers, so check what actually connects to
  the `oireachtas` database first. Not actioned — deliberately left out of the DNS work
  rather than bundled into an unrelated change.

## P2 — confirmed, needs a decision

* **Now that the Thunderbolt link actually works, decide whether SMB moves onto it.**
  The link was repaired 2026-07-26 (see [Thunderbolt link](/network/thunderbolt-link.md))
  and measures **311 MB/s host→Mac against 116 MB/s over the saturated 1GbE** — the reason
  the TB path was wanted in the first place. Two things make this a decision rather than a
  task. First, the docs' standing position is "TB is host-level only, SMB goes over
  Ethernet", adopted after the NAT-over-TB attempt stalled transfers at 85KB — **but that
  attempt was built on an interface that never came up** (`vmbr1` had `bridge-ports none`
  because `nic0` did not exist), so the old verdict rests on a broken foundation and the
  85KB symptom did not reproduce in either direction today. Second, it is not a config
  tweak: [nas (CT100)](/containers/nas.md) is an unprivileged LXC with no interface on the
  link, and the `/30` has no spare address, so it means re-addressing TB to a larger
  subnet, bridging `nic0` into a `vmbr1` that this time has a real uplink, adding a `net1`
  to CT100, and pointing the Mac's SMB mount at that address — while keeping the Ethernet
  path working as the fallback, since TB drops on every cable event. Decide: leave SMB on
  Ethernet (simple, ~116 MB/s), or spend the complexity for roughly 2.5–3× on large media
  copies. Not started — deliberately, since it re-opens a design Dave previously closed.

* **AIVault is 62% full, and ~200G of that is a reservation holding 19.7M of data.**
  Found 2026-07-26 while checking pool headroom before adding Kokoro. `vm-103-disk-0`
  (docker-stack's 200G data disk, wired in on 2026-07-25) is **thickly provisioned**
  because the AIVault storage is `sparse 0` in `/etc/pve/storage.cfg`, so ZFS holds a
  `refreservation` of 203G against `referenced` of 19.7M. Nothing is broken — the
  reservation is what guarantees the VM can't be starved — but it permanently caps the
  Ollama model store at the remaining **164G**, and makes the pool read as nearly full.
  Decide between: (a) leave it, accepting 164G as the real ceiling; (b) shrink the volume
  to something closer to actual need (Postgres+Qdrant are using 47M, so 200G is wildly
  oversized) and reclaim ~150G+; (c) set the storage to `sparse 1` so future disks are
  thin-provisioned. (b) and (c) both touch a live VM disk, so neither was done
  unilaterally. See
  [AIVault](/storage/aivault.md#the-pool-is-62-full-but-nearly-all-of-it-is-a-reservation-found-2026-07-26).

* **The iGPU's BIOS UMA frame buffer is 32GB, above the documented 24GB "sweet spot" —
  and that's the real explanation for the RAM dispute below, not an error.** Checked while
  reconciling the RAM figure (2026-07-25): `dmesg | grep -i 'VRAM:'` shows `VRAM: 32768M`,
  not the "2GB" the GPU playbook previously (and wrongly) said — someone applied a BIOS
  change at some point without updating the doc. 32GB was already flagged in that same
  playbook as "defensible but marginal past 24" *before* this was checked. Right now it's
  costing every other guest 8GB of headroom versus the 24GB target, for no measured
  inference benefit over 24GB (only a 30B-class model needs the extra 8GB, and the doc
  says 30B is still "slow" regardless — 7–14B q4 is called the pleasant range). Decide:
  dial it back to 24GB (reclaims 8GB for the rest of the fleet) or leave at 32GB if
  there's a reason for the extra headroom not currently documented. **Asked 2026-07-25 —
  Dave wants to revisit later, no BIOS change yet.** Requires a full host
  reboot into BIOS — I can't do this remotely, needs Dave at the console. See
  [GPU passthrough & Ollama on Vulkan](/playbooks/gpu-passthrough-ollama-vulkan.md#the-real-performance-lever-bios-uma-frame-buffer).

## P3 — open

* **`/opt/AdGuardHome` on [CT109](/containers/adguard.md) is mode `0755`, and it holds the
  admin bcrypt hash plus eleven backup copies of it.** Surfaced 2026-07-28 during the
  password reset — AdGuard logs it itself on every start: `permcheck: warning: found
  unexpected permissions type=directory path=/opt/AdGuardHome perm=0755 want=0700`.
  Pre-existing, not introduced by that work. Practical risk today is low: CT109 is a
  single-purpose unprivileged LXC with no other human accounts, so "world-readable" means
  little in practice — this is filed for tidiness and because AdGuard has an opinion, not
  because it's believed to be exposed. Fix is `pct exec 109 -- chmod 0700
  /opt/AdGuardHome`; verify the service still starts, since permcheck is the thing that
  cares. Bundle it with the cleanup below.

* **Eleven `AdGuardHome.yaml.bak-*` files have accumulated in `/opt/AdGuardHome`**, one per
  config-editing session since 2026-07-25 (`bak-20260725`, `-20260725b`, `bak2-20260726`,
  `-20260726`, `-20260726-abs`, `-20260726-coder`, `-20260728`, `-20260728-pwd`, …). The
  backup-first habit is right and shouldn't change; the naming is what's drifted, with
  three mutually inconsistent suffix schemes for the same day. Keep the newest two, bin the
  rest, and settle on one scheme. Note each of these contains a **valid historical bcrypt
  hash** — the pre-2026-07-28 ones now hold a superseded password, but they should still be
  removed rather than left readable, which is why this and the item above want doing in one
  pass.

* **`192.168.0.25` sits inside the documented static range and belongs to Dave's Mac
  mini** — identified 2026-07-26, downgraded from "unidentified device" but not closed.
  It was found while allocating an address for
  [audiobookshelf (CT112)](/containers/audiobookshelf.md): `.25` was the obvious next
  static after CT111's `.24`, and it answered `ping`. The MAC `5c:1b:f4:88:46:08` turned
  out to be the **Mac mini's `en0`** — confirmed directly on the Mac during the Thunderbolt
  work (`ifconfig en0` → that MAC, `inet 192.168.0.25`). Not a stranger, but still a
  concrete instance of the
  [open DHCP-conflict problem](/network/ip-addressing.md#the-dhcp-conflict-problem-open):
  the Mac is picking up a DHCP lease inside the static block, so the next person to "take
  the next free IP" still hits it, and the lease can move. CT112 was given `.26`, so
  nothing is broken today. Decide: give the Mac a router DHCP reservation (in or out of the
  static block, but *documented*), and/or shrink the router's DHCP pool to `.100–.200`,
  which prevents the whole class and is still not actioned.

* **Two stale `ward@Davids-Mac-mini` keys still grant root on the host, and nobody knows
  what holds them.** Found 2026-07-26 while fixing SSH: `/etc/pve/priv/authorized_keys`
  carries ED25519 keys `SHA256:/ReD2UrmLm7fx4MlUUf16cm9E7TJ+7BNm1unUf0fbGM` and
  `SHA256:Ux+rd8C64s7HrOQs4/KDdZytTcXd7ousvEGwfZx45x8`, both commented with Dave's
  hostname, and **neither matches any key currently on the Mac** — the Mac holds
  `zcmOLcbLbSIw…` (`id_ed25519`) and `RF7vr9mXxdSm…` (`pve-backup`). So they predate the
  current keypairs: an OS rebuild, a regenerated key, or another machine. They were left
  in place rather than removed unilaterally, because a backup job or a second device could
  legitimately depend on one.
  Decide: identify what they belong to, or remove both now that key auth from the Mac is
  confirmed working. Removal is a two-liner against the backup taken the same day
  (`/root/authorized_keys.bak-20260726`). Until then, two unaccounted-for credentials have
  root on the only host in the lab.

* **Going via Edge sends the entire text of every book to Microsoft.** Not a bug and not a
  reason to avoid it — Dave picked it deliberately for the accent — but it inverts the
  design premise of the whole pipeline, which put Kokoro on-box specifically so no book
  text leaves the LAN. Worth a conscious re-decision if Irish stops being a one-off and
  becomes the default. **Still no local Irish alternative** — Piper was installed
  2026-07-26 and re-checked against its live voice index: 38 locales, English limited to
  `en_GB`/`en_US`, no `en_IE` and no `ga`. So the local side now has two backends and
  neither can do the accent; the trade-off is unchanged, only better understood. See
  [Piper](/playbooks/epub-to-audiobook.md#piper--the-fastest-local-option).

## P4 — housekeeping

* **The Thunderbolt link is running at MTU 1500 and has never been tuned.** Both ends are
  at the Ethernet default while the Linux side reports `maxmtu 65522`; jumbo frames are the
  obvious next lever on a point-to-point link with no switch in between, and the measured
  ~300 MB/s is bounded by SSH crypto and `nc`, not by the wire, so the real headroom is
  unknown. Not attempted because **the Mac side needs `sudo`** (`sudo ifconfig bridge0 mtu
  9000`, and permanently via Network settings), which isn't available non-interactively —
  and both ends must change together or the mismatch blackholes large packets. Do it with
  Dave present, host side first (`ip link set nic0 mtu 9000` plus an `mtu` line in the
  `/etc/network/interfaces` stanza), and re-measure before keeping it. See
  [Thunderbolt link](/network/thunderbolt-link.md).

* **code-server runs as root in CT113, and its login page is also served unencrypted on the
  LAN.** Neither is a bug; both are choices worth a conscious nod. The service runs as
  **root inside the container** via `code-server@root`, so the login password is
  root-in-CT113 including its terminal — defensible in an unprivileged LXC where
  container-root is host UID 100000, and near-necessary for a dev box that needs `apt`, but
  switch to `code-server@<user>` if that stops being true. And `cert: false` means
  `http://192.168.0.27:8080` answers in clear on the LAN, same as Jellyfin/SABnzbd/Grafana
  and for the same reason (no active Proxmox firewall — the LAN boundary is the model). Use
  `https://coder.lan`. The password itself **was set 2026-07-26** and the config tightened to
  `0600`; change it with the one-liner on
  [code-server (CT113)](/containers/code-server.md#config-and-password).

* **`opencode.lan` is a dead proxy entry** — it resolves and then fails, rather than not
  resolving. Found 2026-07-26 while adding `coder.lan`. The
  [Caddy playbook](/playbooks/reverse-proxy-caddy.md) listed it under "not proxied — nothing
  listens", but the Caddyfile has had an `opencode.lan → 192.168.0.18:8080` block all along
  and AdGuard has a matching rewrite; only the *backend* is absent (re-confirmed: `curl` to
  `192.168.0.18:8080` from the host times out). Harmless today. Either drop both entries, or
  point them at whatever on [CT107](/containers/opencode.md) is actually meant to be
  reachable. The playbook's text has been corrected either way.

* **CT111 has no root password and no SSH key**, so it's reachable only via
  `pct exec 111` from the host or the Proxmox web console. That's sufficient for how it's
  used (batch jobs kicked off from the host) and avoids creating a credential nobody
  asked for — but if you want to drive it directly, set one yourself with
  `pct exec 111 -- passwd`. See [audiobooks (CT111)](/containers/audiobooks.md).

* **Three test audiobooks are sitting in the library**, all left deliberately, all safe to
  delete whenever. Together they're a side-by-side audition of the three backends, which is
  the only reason to keep them:
  * `Audiobooks/rory-test/0005_2.mp3` (14MB, ~14.7 min, one chapter of *RORY* by Alan
    Shipnuck) — the original Kokoro quality check, kept so the voice can be judged before
    committing to a full book.
  * `Audiobooks/edge-ie-smoketest/` (2026-07-26) — a few seconds of synthetic text in
    `en-IE-ConnorNeural`, the artifact of the Edge verification. Serves as a reference
    sample of the Irish voice if you want to hear it before picking a narrator; otherwise
    it's litter. Note it was written **twice** by a misfiring test script; the second run
    overwrote the first, so there's only one file.
  * `Audiobooks/piper-smoketest/` (2026-07-26, two files, 64KB) — the Piper verification,
    in `en_GB-cori-medium`. Worth keeping until Piper's voice quality has actually been
    judged against Kokoro's, since that's the open question about whether the 4× speed
    advantage is usable.

  **This got more visible on 2026-07-26**: [audiobookshelf (CT112)](/containers/audiobookshelf.md)
  now serves this directory, and its scanner will present all three — plus `logs/` and
  `audiobook_output/` — as library items. They're no longer just files on a pool, they're
  entries in a UI Dave will actually look at. Clearing the ones that have served their
  purpose before the first library scan is the cheap moment to do it.

* **Two superseded backup scripts on CT111 from the Piper install**, both 2026-07-26, both
  safe to delete: `/opt/piper/piper.bak-20260726` (the adapter before it learned to drop
  `None`-valued flags) and `/usr/local/bin/audiobook-piper.bak-20260726` (the wrapper
  before the absolute-path fix). Kept only until Piper has run a full-length book.

* **Piper's voice quality has not been judged.** It's ~4× faster than Kokoro (~20× realtime
  vs ~4.5–6×), which would take a 10-hour book from 2 hours to 30 minutes — but it's a much
  smaller model and nobody has listened to a real chapter yet, only smoke tests. If it
  holds up it should probably become the default local backend; if it doesn't, the speed
  number is irrelevant. Listen to `Audiobooks/piper-smoketest/` and decide. See
  [Speed](/playbooks/epub-to-audiobook.md#speed).

* **Two small AIVault datasets are now genuinely obsolete.** `AIVault/postgres` and
  `AIVault/models` (96K each) were leftovers from the original unwired disk plan, now
  superseded by the real bind-mount migration below. Low-priority cleanup, whenever
  convenient: `zfs destroy AIVault/postgres AIVault/models`.

* **Two `.old-20260725` backup copies from the Postgres/Qdrant migration are still on
  docker-stack's root disk** (`/data/postgres.old-20260725`, `/data/qdrant.old-20260725`
  — 47M total). Kept deliberately as a safety net; safe to delete once the migration's
  been running cleanly for a while. See
  [AIVault](/storage/aivault.md#postgresqdrant-migrated-here-2026-07-25).

* **Change the temporary Grafana admin password.** Reset to `N5proTemp2026xyz` on
  2026-07-25 to unblock dashboard creation via the API (the original password was
  unknown, not the Grafana default). Change it at
  `https://grafana.lan/profile/password` when convenient — only you should do this step.
  See [grafana (CT110)](/containers/grafana.md#admin-credentials--reset-2026-07-25-needs-daves-attention).

* **All three ZFS pools have `zpool upgrade` available but not run** (older on-disk
  feature set, likely never touched since the TrueNAS import). Not urgent, not
  investigated further — flagging in case it matters for a future feature.

* **Trust the Caddy internal CA on your devices — now optional, and probably unnecessary.**
  `https://jellyfin.lan` and friends serve real (self-signed) HTTPS via the reverse proxy
  set up 2026-07-25, and browsers show a cert warning until each device trusts the root CA.
  **As of 2026-07-28 there's a better route**: the same services also answer on
  `https://<name>.133gsl.ie` with a **publicly-trusted Let's Encrypt wildcard**, which needs
  no per-device trust step at all — see
  [133gsl.ie on Cloudflare DNS](/playbooks/dns-cloudflare-133gsl-ie.md). Use the `.ie` names
  and this item goes away. The `.lan` names are being kept in parallel as a fallback for
  now, so the internal-CA option remains valid if you want it; the retrieval command is in
  [Reverse proxy via Caddy](/playbooks/reverse-proxy-caddy.md#trusting-the-internal-ca).

* **Decide whether the `.lan` names get retired.** Both `.lan` (internal CA) and
  `.133gsl.ie` (Let's Encrypt) now front the same 14 services through the same Caddy
  instance. Deliberately left running in parallel from 2026-07-28 so there's a fallback if
  the Cloudflare token or zone breaks. Revisit after a week or two of the `.ie` names being
  used in anger: either retire `.lan` and halve the config surface, or keep both and accept
  that every new service needs adding twice. Note the `.ie` set deliberately **omits
  `opencode`** (dead backend, see the item above) — don't treat that as an oversight when
  comparing the two tables.

* **DHCP pool still overlaps the static range** (long-standing, pre-dates this review).
  Recommendation: shrink router DHCP pool to `.100–.200`. Not yet actioned. See
  [IP addressing](/network/ip-addressing.md#the-dhcp-conflict-problem-open).

# Citations

[1] Findings from direct Proxmox UI / HAOS console review (Dave's homelab, 2026-07-25).
