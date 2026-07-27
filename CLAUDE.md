# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not a software project** — there is no code, no build, no lint, no tests. It's a
markdown knowledge bundle documenting Dave's homelab: a single Minisforum N5 Pro running
Proxmox that handles NAS, AI inference, database, media, and smart-home duty. There is no
fleet, no second host, no HA cluster — all advice and commands should be scoped to this one
machine, not generic Proxmox guidance.

The bundle was originally extracted from the `n5-pro-homelab` Claude Skill (working
knowledge accumulated across troubleshooting sessions) and is maintained here going forward.
Start at [index.md](index.md) for the top-level map.

## Structure

Hub-and-spoke: every directory has an `index.md` that links out to topic pages.

- `host/n5-pro.md` — the physical host: hardware, subnet, working-style notes.
- `containers/` — one file per LXC/VM (14 guests), plus `index.md` listing all of them with CT/VM IDs and IPs.
- `storage/` — one file per ZFS pool (MediaTank, AIVault, DataPool).
- `network/` — IP addressing scheme, AdGuard DNS/rewrite table, Thunderbolt link.
- `playbooks/` — step-by-step procedures; `troubleshooting-gotchas.md` is the fast-path checklist to check *before* first-principles debugging.
- `actions.md` — standing, prioritised (P0–P4) list of confirmed issues/follow-ups not yet actioned, with a `## Resolved` section for closed items.
- `log.md` — append-only chronological changelog of what changed in the bundle and why, newest at top, entries numbered per day (`## 2026-07-25 (7)`, `(6)`, …).

## Page conventions

Topic pages (not `index.md` files) open with YAML frontmatter:

```yaml
---
type: Service | ZFS Pool | Playbook | Reference   # varies by content
title: Human-readable title
description: One-sentence summary
resource: /SomePath                                # optional, e.g. mountpoint
tags: [lowercase, keywords]
timestamp: 2026-07-19T00:00:00Z
---
```

Body uses `#`/`##` headers, ends with a `# Citations` section pointing back to the source
(usually the `n5-pro-homelab` Claude Skill, or "direct \[UI/console\] review" with a date for
findings verified live). Links are **relative within the same directory**
(e.g. `containers/index.md` → `[nas (CT100)](nas.md)`) and **absolute from repo root when
crossing directories** (e.g. `[IP addressing](/network/ip-addressing.md)` from a containers
page).

## Keeping the bundle current

When a troubleshooting session on the actual host uncovers new facts or fixes something:

1. Update the relevant topic page(s) with the verified state.
2. Log the change in `log.md` as a new dated/numbered entry (prose, explain what was done
   and why, link every touched page).
3. Reflect the issue in `actions.md`: add it under the right priority if newly found, or
   strike it through and move it to `## Resolved` with the fix details if closed in this
   session. Don't let findings live only in `log.md` history — `actions.md` is the
   standing source of truth for what's still open.

## Working style with Dave

- Dave is a **cloud engineer (Google Cloud PSO)** — skip 101-level explanations of infra
  concepts (containers, systemd, etc.). He's newer to home networking/NAS specifics than to
  cloud, so don't assume the same depth there.
- During active troubleshooting on the host: **commands only, no preamble.** Lead with the
  command; explain only if asked or a step is genuinely dangerous.
- **Verify live state before asserting it** — `systemctl show`, `cat` the real config file,
  `ls -ln` the actual device — rather than trusting an editor view or assumed default. Past
  sessions hit dead ends this way.
- When a change touches an unprivileged LXC, flag any bind mount whose ownership could
  shift under the idmap before rebooting (unprivileged root maps to host UID **100000**;
  `chown` must use that mapped UID on the **host**, never `1000` from inside the container).

## Key facts that shape all advice

- Subnet is **192.168.0.x/24** (not `.1.x`), gateway `192.168.0.1`. Static guest IPs live in
  `.11–.23`.
- IPv6-off is the *documented* policy for guests but is **not actually enforced** on most of
  them as of the last audit — check current state rather than assuming the policy holds; see
  `network/ip-addressing.md`.
- Thunderbolt (`10.10.10.2/30`, Mac at `10.10.10.1`) is host-level only (ZFS sends, VM disk
  transfers) — SMB to the Mac goes over standard Ethernet, not TB. It should appear as
  `nic0`; **if you ever see `thunderbolt0`, the rename has failed and the link is dead** —
  that was the state from install until 2026-07-26, see `network/thunderbolt-link.md`.
