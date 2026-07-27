---
type: LXC Container
title: code-server (CT113)
description: code-server (VS Code in the browser), unprivileged LXC at 192.168.0.27, served at https://coder.lan.
tags: [proxmox, lxc, code-server, vscode, ide, development]
timestamp: 2026-07-26T00:00:00Z
---

CT113, hostname `code-server`, unprivileged LXC at **192.168.0.27**. Created 2026-07-26 to
give Dave a browser-based VS Code on the box, reachable at **`https://coder.lan`**.

**This is `code-server`, not Coder.** The two are different products from the same vendor
and the request named both ("Coder Code Server"). Installed here is
[coder/code-server](https://github.com/coder/code-server) — a single binary that serves one
VS Code instance over HTTP. It is *not* [coder/coder](https://github.com/coder/coder), the
multi-user workspace platform, which needs Postgres and a Docker/Kubernetes provisioner to
hand out workspaces and would have been a substantially bigger build. If what's actually
wanted is the platform, this container is the wrong thing and should be replaced rather
than extended.

# The hostname and the domain deliberately don't match

Container hostname is `code-server`; the DNS name is **`coder.lan`**, as requested. Every
other guest in the fleet has `<hostname>.lan`, so this is the one place that convention
breaks. Noted here so a future reader grepping for `coder` in `pct list` doesn't conclude
the container is missing.

# Spec

Debian 13 (trixie) from `local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst`, 4 cores,
4GB RAM, 512MB swap, 32G rootfs on [DataPool](/storage/datapool.md) (757M used after
install). `onboot: 1`, `features: nesting=1`, `searchdomain: lan`, unprivileged. Static
`.27`, DNS at [AdGuard](adguard.md) (`192.168.0.20`), IPv6 disabled via
`/etc/sysctl.d/99-disable-ipv6.conf` **at provisioning time** — built to
[fleet policy](/network/ip-addressing.md) from the start, following CT111/CT112's
precedent.

Sized deliberately larger than the recent media containers: 4 cores / 4GB because a
language server plus a build is the normal workload, and a 32G rootfs (vs CT112's 16G)
because checkouts, `node_modules` and toolchains accumulate. There are **no bind mounts** —
everything lives on the rootfs. If projects should live on a pool instead, add an `mp0`
then rather than growing the rootfs.

Verified to survive `pct reboot`: unit back `active`, still listening on `0.0.0.0:8080`,
IPv6 still zero addresses, `https://coder.lan/login` still `200`.

# Why `.27`

Next free static above [audiobookshelf (CT112)](audiobookshelf.md)'s `.26`. **Ping-checked
before allocating** (`.27` and `.28` both silent from the host, no ARP entry) — the ledger
listed `.27`–`.32` as free from the 2026-07-26 sweep, but that page is explicit that the
ping is not a formality, and `.25` is the standing proof. `.15` remains the one gap inside
the `.11`–`.24` block and was left alone for the same collision-zone reason CT111 and CT112
built upward.

# Install — the official one-liner was not used

code-server **4.130.0** (bundling Code 1.130.0), from the project's own `.deb` on GitHub
Releases:

```
https://github.com/coder/code-server/releases/download/v4.130.0/code-server_4.130.0_amd64.deb
sha256  2df0f7718a1e6ac090fa39226c1a291453403e3ca2e636804695648cdb24a851
```

Upstream's documented path is `curl -fsSL https://code-server.dev/install.sh | sh`. That
URL is real (unlike the Audiobookshelf case in
[CT112](audiobookshelf.md#install--the-official-one-liner-was-not-used)), but it just
detects the platform and fetches this same `.deb`, so the pipe-to-shell step buys nothing
here and was skipped. **Upstream publishes no checksum file** for the release — the assets
are the packages and tarballs only — so the hash above is a *record of what was installed*,
not a verification against a published value. The only integrity guarantee at install time
was HTTPS to github.com. Compare against that hash if the container is ever rebuilt.

# It runs as root, on purpose

Started via the deb's own systemd template unit: `systemctl enable --now code-server@root`.

The alternative was a dedicated service user. Root was chosen because this is a
single-user dev box inside an **unprivileged** LXC — container-root maps to host UID
`100000` and holds no host privilege — and because the first thing a dev container gets
asked to do is `apt install` a toolchain, which a non-root service user can't. The
trade-off is real and local: anything with the login password gets root *in this
container*, including its terminal. If that ever stops being acceptable, switch to
`code-server@<user>` and move `~/.config/code-server/config.yaml` with it.

A `/root/workspace` directory exists as a default place to put checkouts; nothing enforces
its use.

# Config and password

`/root/.config/code-server/config.yaml`, mode **0600** (the deb ships it `0644`):

```yaml
bind-addr: 0.0.0.0:8080
auth: password
password: <20-char random, set 2026-07-26 — given to Dave directly, not recorded here>
cert: false
```

Only `bind-addr` was changed from the installed default (`127.0.0.1:8080`) — Caddy runs on
a *different* guest (`192.168.0.14`), so loopback-only would have been unreachable through
the proxy.

`cert: false` is correct here and not an oversight: TLS is Caddy's job, same as every other
service in this fleet. The consequence is that `http://192.168.0.27:8080` is also reachable
**unencrypted on the LAN**, carrying the login password in clear. That's identical to
Jellyfin, SABnzbd, Grafana and the rest — there's no Proxmox firewall active anywhere on
this host (see [n5-pro](/host/n5-pro.md#network)), so the LAN boundary is the whole
security model. Use `https://coder.lan`, not the IP.

The deb's install-time random password was **replaced 2026-07-26** at Dave's request with a
fresh 20-character random string (`openssl rand -base64 24`, generated in-container), and
the file tightened to `0600` — it shipped `0644`, which is world-readable, and while the
container has no other users today that's a needless default to keep. Previous config kept
as `config.yaml.bak-20260726`, also `0600`.

The value is **deliberately not written down in this bundle** — it was handed to Dave in the
session that set it. To change it:

```bash
pct exec 113 -- sed -i 's/^password:.*/password: your-new-password/' \
  /root/.config/code-server/config.yaml
pct exec 113 -- systemctl restart code-server@root            # required — config is read at start
```

Storing an argon2 `hashed-password:` instead of the cleartext is supported by code-server
and would be the upgrade if this ever holds anything sensitive; it was skipped to keep the
one-line change procedure above working.

# Reverse proxy

`coder.lan` → `192.168.0.27:8080`, via
[Caddy on docker-stack](/playbooks/reverse-proxy-caddy.md). Plain `reverse_proxy`, no
`transport` block — code-server is entirely websocket-driven once the IDE loads, and
Caddy 2 upgrades those transparently, the same as Audiobookshelf's socket.io and unlike
`proxmox.lan`. The AdGuard rewrite answers `192.168.0.14` (docker-stack), not `.27`; see
the note in [DNS via AdGuard](/network/dns-adguard.md) on why a block of identical `.14`
answers is correct by design.

# Verification — including the websocket

Checked 2026-07-26, in this order, all through Caddy under real DNS unless noted:

| Step | Result |
|---|---|
| `dig coder.lan @192.168.0.20` | `192.168.0.14` |
| `GET /login` | `200` |
| `POST /login` with the password | `302` → `/`, sets a `code-server-session` cookie (itself an argon2id token, not the password) |
| `GET /` with that cookie | `200`, 4KB of workbench HTML — `workbench.js`, `remoteAuthority`, `productConfiguration`; **not** the login page |
| All 6 assets the workbench pulls | `200`, including the 17.5MB `workbench.js` bundle and the 1.3MB CSS |
| **Websocket upgrade** | **`101 Switching Protocols`, `Server: Caddy`** |
| **VS Code remote protocol** | server's first frame is a 13-byte binary control message, **type 9 = `Resume`** — what a live VS Code server sends to tell the client to start streaming |
| Same two steps direct on `192.168.0.27:8080` | identical, so the protocol works with and without the proxy |

The websocket was the open question from the build and it is now closed: a plain
`reverse_proxy` with no `transport` block carries it, matching Audiobookshelf's socket.io
and unlike `proxmox.lan`. The grey-screen-that-never-finishes failure mode is ruled out.

**What was not done: loading it in a browser.** That needs the password typed into the
login form, which isn't something to do on Dave's behalf, and `https://coder.lan` also hits
Chrome's interstitial until the device trusts
[Caddy's internal CA](/playbooks/reverse-proxy-caddy.md#trusting-the-internal-ca) — the
standing per-device action in [actions.md](/actions.md). Everything the browser would
fetch, and the protocol it would then speak, has been exercised directly instead.

# Citations

[1] Direct build and live verification on this host, 2026-07-26 — container creation,
install, Caddy/AdGuard wiring, an end-to-end `https://coder.lan` check, and a
reboot-survival test.
[2] code-server v4.130.0 release assets, https://github.com/coder/code-server/releases
(retrieved 2026-07-26).
