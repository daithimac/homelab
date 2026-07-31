---
type: Playbook
title: n8n librarian agent
description: Design and implementation plan for an autonomous n8n agent that curates the Calibre, inbox and audiobook libraries on MediaTank, via a purpose-built Librarian API on CT111.
tags: [n8n, agent, calibre, audiobooks, ebooks, automation, ollama, mediatank, lxc]
timestamp: 2026-07-31T00:00:00Z
---

> **Status: designed, not built.** Nothing in this page has been run on the host. Every
> path, version and library layout marked **VERIFY** is an assumption drawn from the rest
> of this bundle, not something read off the box. Work through
> [Before you start](#before-you-start--verify-these-six-things-on-the-box) first; if a
> check comes back different, fix this page before building against it.

A single n8n workflow that curates three libraries nightly, unattended: the Calibre ebook
library, the SMB drop `Books/inbox`, and the audiobook tree that
[Audiobookshelf (CT112)](/containers/audiobookshelf.md) serves. It files new books,
enriches metadata, stacks duplicate formats, keeps the audiobook tree in the layout
Audiobookshelf wants, and queues ebooks that have no audiobook yet through the existing
[EPUB/PDF to audiobook](epub-to-audiobook.md) pipeline.

# Where the pieces run, and why

The awkward fact that shapes the whole design: **n8n cannot see the library.** It runs in
Docker on [docker-stack (VM103)](/containers/docker-stack.md), and a VM cannot bind-mount
a ZFS dataset — the same constraint that put the audiobook converter in an LXC rather than
next to n8n in Compose ([CT111](/containers/audiobooks.md#storage)). So n8n is the brain
and holds no filesystem access at all; a small **Librarian API on
[audiobooks (CT111)](/containers/audiobooks.md)** does every operation that touches a file.

| Piece | Where | Role |
|---|---|---|
| **Librarian API** | CT111, `192.168.0.24:8099` | The only thing that touches files. Owns every guardrail. |
| **n8n workflow + agent** | VM103, `192.168.0.14:5678` | Schedule, LLM reasoning, retries, notification. |
| **Ollama** | CT102, `192.168.0.13:11434` | The agent's model. See [the model gap](#the-model-gap--nothing-installed-can-drive-a-tools-agent). |
| **Postgres** | VM103, `192.168.0.14:5432` | Action ledger + catalogue snapshot. Already exists for n8n. |
| **Audiobookshelf** | CT112, `192.168.0.26:13378` | Rescan trigger + "which books already have audio". |
| **ZFS snapshots** | host, systemd timer | Pre-run rollback point. Only the host can do this. |

CT111 is the right home for the API and not an arbitrary one: it already has **calibre
8.5.0** (so `calibredb` is present, not just `ebook-convert`), **ffmpeg 7.1.5**, Python
3.13.5, and — the part that matters — `mp0: /MediaTank/media,mp=/srv/media`, the *whole*
media tree rather than CT112's deliberately narrow `Audiobooks`-only mount. It can also
launch conversions locally with `systemd-run`, so queueing an audiobook needs no `pct exec`
from outside and nothing gets hypervisor access.

**Do not put this behind Caddy.** In-fleet service-to-service calls in this homelab go to
raw `IP:port` — openwebui and sillytavern hit `192.168.0.13:11434` directly rather than
`ollama.lan`, for the stated reason that proxying adds a hop and a TLS handshake for
nothing ([ollama (CT102)](/containers/ollama.md)). n8n calls `http://192.168.0.24:8099`.
That also means **no new Caddyfile block and no new AdGuard rewrite** — skip
[Adding a new domain](reverse-proxy-caddy.md#adding-a-new-domain) entirely. Only revisit if
you later want the HTML report readable from a browser.

# Before you start — verify these six things on the box

Everything downstream depends on these, and this page guessed at four of them.

```bash
# 1. WHERE IS THE CALIBRE LIBRARY? This page assumes /srv/media/Books/Calibre Library.
pct exec 111 -- find /srv/media/Books -maxdepth 3 -name metadata.db

# 2. Is calibredb actually on PATH? (the bundle only ever names ebook-convert)
pct exec 111 -- /usr/bin/calibredb --version
pct exec 111 -- ls /usr/bin/fetch-ebook-metadata

# 3. What are the three trees really called, and how big?
pct exec 111 -- ls -la /srv/media/Books /srv/media/Audiobooks

# 4. Is MediaTank/media a dataset (snapshottable on its own) or just a directory?
zfs list -r -o name,used MediaTank

# 5. Does anything else write the Calibre library concurrently?
#    (no Calibre server or Calibre-Web is documented anywhere in this bundle — confirm)
pct exec 111 -- ss -ltnp | grep -E '8080|8083'

# 6. Audiobookshelf must be initialised before its API answers — it reported
#    "isInit": false as of 2026-07-26 and creating the admin account is Dave's job.
curl -s http://192.168.0.26:13378/status
```

Check 1 is the one that changes this page most. If the library sits somewhere else, or if
what you thought was a Calibre library has no `metadata.db`, stop and re-read
[the Calibre rules](#calibre-is-a-database-not-a-folder) — the whole safety model assumes a
real library.

Two prerequisites from [actions.md](/actions.md) are worth closing first rather than
working around: **Audiobookshelf's first-run setup** (P-item, needs the admin account and a
library pointing at `/audiobooks`), and the **leftover test artefacts** in
`/srv/media/Audiobooks` — `rory-test/`, `edge-ie-smoketest/`, `piper-smoketest/`,
`audiobook_output/`, `logs/`. Clear those by hand before the first run, or the librarian's
first report will be a page of findings about your own smoke tests.

# The safety contract

Fully autonomous file operations against a 1.4T library is the mode that can quietly
destroy it, and "the LLM decided to run `calibredb remove`" is exactly how. The autonomy is
real — no approval step, nothing waits for Dave — but **every guard below lives in the API,
not in the prompt.** A prompt is a request; the API is the thing that actually refuses. An
agent that hallucinates a path or loops on a retry hits a wall made of code.

1. **Nothing is ever unlinked.** There is no delete endpoint. `trash` moves the item to
   `/srv/media/.librarian/trash/<run-id>/`, preserving relative path. A separate host-side
   job prunes trash older than 30 days — that is the only thing in the system that calls
   `rm`, it runs on a timer, and no agent can invoke it.
2. **A ZFS snapshot exists before every run.** Taken host-side at 02:50, ten minutes ahead
   of the 03:00 workflow. A bad night is `zfs rollback`, not archaeology.
3. **Hard path allowlist**, enforced by `os.path.realpath` after resolution, not by string
   prefix — symlinks and `..` both get normalised away before the check. Three roots only.
   Anything else is a 403 regardless of what the agent asked for.
4. **A per-run mutation budget.** The API counts mutating calls against a run ID and starts
   returning `429` past the cap (start at 50). A correct night uses a handful; a runaway
   loop hits the ceiling in a minute and stops there. This is the single most valuable
   guard, because it bounds damage from *any* failure mode including ones nobody predicted.
5. **Single writer.** Every `calibredb` call takes an exclusive `flock` on the library
   directory. Also set the n8n workflow to **not** run concurrently, so a long night can't
   overlap the next one.
6. **Append-only ledger.** Every mutation writes `(run_id, ts, op, target, before, after)`
   to Postgres before it happens and updates on completion. Inverse-replayable — you can
   reconstruct exactly what a run did without diffing 1.4T.

## Calibre is a database, not a folder

`metadata.db` is the truth; the `Author/Title (id)/` tree is derived output. Move a file in
there by hand and Calibre still believes the old path — the record silently points at
nothing, and it stays broken until someone notices a book won't open. So the API exposes
**no generic file operation against the Calibre root at all.** `mv`, `rename` and `trash`
are wired to reject that root outright; the only way in is the `calibre_*` endpoints, which
shell out to `calibredb`. That asymmetry is deliberate and it is the reason the inbox is a
separate tree.

# The Librarian API

FastAPI on CT111, `0.0.0.0:8099`, bearer-token auth. Debian 13 is PEP-668
externally-managed — **a venv is required, not a preference**, exactly as CT111's other two
Python installs already are:

```bash
pct exec 111 -- python3 -m venv /opt/librarian/venv
pct exec 111 -- /opt/librarian/venv/bin/pip install fastapi uvicorn ebooklib mutagen psycopg[binary] httpx
```

## Endpoints

Read endpoints are free; mutating ones (marked ✱) count against the run budget and write
the ledger.

| Endpoint | Purpose |
|---|---|
| `POST /run/start` → `{run_id}` | Opens a run. Records the snapshot name. Resets the budget. |
| `POST /run/end` ✱ | Closes the run, returns the ledger for the report. |
| `GET /scan/inbox` | Lists `Books/inbox` with format, size, mtime, and parsed OPF metadata. |
| `GET /scan/audiobooks` | One entry per book dir: track count, total duration, ID3 tags, cover present, **zero-byte files**. |
| `GET /calibre/list` | `calibredb list --for-machine -f all` — the whole catalogue as JSON. |
| `GET /calibre/search?q=` | `calibredb search`, for duplicate checks. |
| `POST /calibre/add` ✱ | `calibredb add --automerge=ignore`. Returns the new/matched book id. |
| `POST /calibre/set_metadata` ✱ | `calibredb set_metadata` for one book id. |
| `POST /calibre/fetch_metadata` | `fetch-ebook-metadata` against OpenLibrary/Google Books. **Read-only** — returns a candidate, writes nothing. |
| `POST /convert/pdf_to_epub` ✱ | `ebook-convert` into a temp dir, then add as a format. |
| `POST /audiobook/queue` ✱ | `systemd-run --unit=lib-<slug> --collect` wrapping `/usr/local/bin/audiobook`. |
| `GET /audiobook/status/<unit>` | `systemctl is-active` + log tail. |
| `POST /fs/move` ✱ | Move within the audiobook tree or inbox only. Rejects the Calibre root. |
| `POST /fs/trash` ✱ | Move into `.librarian/trash/<run_id>/`. The only removal. |
| `POST /abs/rescan` ✱ | Audiobookshelf `POST /api/libraries/<id>/scan`. |

## The four guards, in code

The rest of the service is boilerplate. These are the parts that are load-bearing and easy
to get subtly wrong:

```python
ROOTS = [Path("/srv/media/Books/inbox").resolve(),
         Path("/srv/media/Audiobooks").resolve(),
         Path("/srv/media/.librarian").resolve()]
CALIBRE_ROOT = Path("/srv/media/Books/Calibre Library").resolve()   # VERIFY

def safe_path(p: str, *, allow_calibre=False) -> Path:
    # realpath FIRST: resolves symlinks and .. before any comparison.
    rp = Path(p).resolve()
    if rp == CALIBRE_ROOT or CALIBRE_ROOT in rp.parents:
        if not allow_calibre:
            raise HTTPException(403, "Calibre library is calibredb-only")
        return rp
    if not any(r == rp or r in rp.parents for r in ROOTS):
        raise HTTPException(403, f"path outside allowlist: {rp}")
    return rp

def budget(run_id: str):                       # guard 4
    n = redis_or_pg_incr(run_id)
    if n > OP_BUDGET:                          # start at 50
        raise HTTPException(429, "run mutation budget exhausted")

def trash(run_id: str, p: str):                # guard 1 — move, never unlink
    src = safe_path(p)
    dst = TRASH / run_id / src.relative_to("/srv/media")
    dst.parent.mkdir(parents=True, exist_ok=True)
    shutil.move(src, dst)

@contextmanager
def calibre_lock():                            # guard 5 — single writer
    with open("/run/librarian.calibre.lock", "w") as fh:
        fcntl.flock(fh, fcntl.LOCK_EX)
        yield
```

`safe_path` resolving before comparing is not pedantry: a plain `str.startswith` check is
defeated by `/srv/media/Audiobooks/../Books/Calibre Library`, which is precisely the shape
of path an LLM produces when it is confused about where it is.

## Ownership — the setgid trap, again

Anything the API creates on the pool inherits CT111's root, which maps to host **100000**.
The library tree is group `100112` mode `2775`, and **the setgid bit is what makes that
work** — without it new files land in CT111's root group and every other media container
loses access, the exact failure already documented for the audiobook output directory. So
create the new directories **on the host**, with mapped UIDs, per
[ZFS bind-mount permissions](zfs-bind-mount-permissions.md):

```bash
# on the HOST — never chown 1000 from inside an unprivileged container
mkdir -p /MediaTank/media/.librarian/trash
chown -R 100000:100112 /MediaTank/media/.librarian
chmod -R 2775 /MediaTank/media/.librarian
```

A dotted directory keeps it out of Audiobookshelf's and Jellyfin's scans. Verify after the
first trash operation that the moved file came out `100000:100112` rather than trusting the
mode bits — this bundle has been caught by that before.

## systemd unit

```ini
[Unit]
Description=Librarian API
After=network-online.target

[Service]
ExecStart=/opt/librarian/venv/bin/uvicorn librarian:app --host 0.0.0.0 --port 8099
WorkingDirectory=/opt/librarian
EnvironmentFile=/etc/librarian.env
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

`/etc/librarian.env` holds `LIBRARIAN_TOKEN`, `PG_DSN` and `ABS_TOKEN`, mode **600**. Not
`664` — see the P1 in [actions.md](/actions.md) about `docker-compose.yml` being
world-readable with a live Gmail app password in it. Don't add a second instance of that
bug. And verify the env actually reached the process rather than trusting the file:

```bash
pct exec 111 -- systemctl show librarian -p Environment
```

# Host-side: snapshot and prune

Two systemd timers on the **host** (CT111 cannot run `zfs`, and giving it the ability would
undo the whole containment argument). Adjust the target if check 4 showed `MediaTank/media`
is a plain directory rather than a dataset — then it's `zfs snapshot -r MediaTank@...`.

```bash
# /usr/local/sbin/librarian-snapshot  — 02:50 daily
zfs snapshot MediaTank/media@librarian-$(date +%Y%m%d)
zfs list -t snapshot -o name -H MediaTank/media | grep '@librarian-' | head -n -14 | xargs -r -n1 zfs destroy

# /usr/local/sbin/librarian-prune-trash  — weekly
find /MediaTank/media/.librarian/trash -mindepth 1 -maxdepth 1 -mtime +30 -exec rm -rf {} +
```

Fourteen snapshots is two weeks of rollback points. They cost almost nothing until the
librarian actually changes something, which is the property that makes this cheap.

# The n8n workflow

```
Schedule Trigger (03:00)
  → HTTP Request      POST /run/start            → run_id
  → HTTP Request      GET  /scan/inbox
  → HTTP Request      GET  /calibre/list
  → HTTP Request      GET  /scan/audiobooks
  → Loop Over Items   one book / one directory per iteration
    → HTTP Request    GET  /calibre/search?q=<this title>   → ≤10 candidates
    → AI Agent (Tools Agent)
        Chat Model:   Ollama gpt-oss:20b          (192.168.0.13:11434)
                      num_ctx 32768, reasoning effort low
        Memory:       none — each item is independent, and shared memory across
                      items is how one bad match contaminates the next
        Tools:        HTTP Request Tool × 9, one per mutating endpoint
        Output:       Structured Output Parser (one item's actions)
  → Aggregate         merge per-item results into the run report
  → Postgres          insert run summary
  → HTTP Request      POST /run/end
  → HTTP Request      POST /abs/rescan            (only if audiobooks changed)
  → Send Email        the report
```

Settings that matter: **Execution Order v1**, **"Do not run concurrently"**, workflow
timeout ~2h, and an **Error Workflow** set — an unattended agent that dies silently is
worse than one that fails loudly.

## Context: batch the work, don't dump the catalogue

**`OLLAMA_CONTEXT_LENGTH=12400` is set globally on CT102** — chosen against a dense model,
per [Local LLM daily driver](local-llm-daily-driver.md#kv-cache-a-memory-lever-not-a-speed-lever).
That is a hard ceiling and Ollama **truncates silently** when you cross it. For an
autonomous agent that is not a performance problem, it is a correctness catastrophe: the
system prompt is ~1,500 tokens, the tool schemas another ~800, and a full `calibre/list`
dump of a few hundred books is 20k+ on its own. Cross the line and the cardinal rules are
what falls out of the window — you get an agent with tools and no constraints, at 03:00,
with nobody watching.

Two changes, both required:

1. **Loop, don't dump.** Put a **Loop Over Items** node around the agent and give it *one
   inbox file per iteration*, plus the ≤10 candidate Calibre records returned by a
   `calibre_search` on that book's title — not the catalogue. Steps 2–4 get their own
   batched loops. This keeps each turn a few thousand tokens, and it is the better agent
   design regardless: small context measurably improves rule adherence, and a per-item loop
   means one bad item fails one iteration instead of poisoning the whole run.
2. **Raise `num_ctx` per request to 32768**, as a request-level option from the n8n Ollama
   node rather than by changing the global env — the global affects openwebui and
   sillytavern too. With `OLLAMA_KV_CACHE_TYPE=q8_0` and flash attention already on, 32K is
   affordable on a ~4B-active MoE; measure the KV cost with the command above before
   assuming it.

Belt and braces: the loop keeps you far under the ceiling, and `num_ctx` means a single
unusually large item degrades instead of silently losing the rules.

## The model — pull `gpt-oss:20b`

**Nothing currently installed can drive a tools agent.** All seven models in
`AIVault/ollama-models` are RP/creative/uncensored finetunes —
`gemma-4-26B-A4B-...-ultra-uncensored-heretic`, `Melody1437`, `MistralRP-Noromaid`,
`Qwen3.5-4B-NSFW-...` ([full inventory](/containers/ollama.md#model-inventory-2026-07-29)).
That class routinely ships a chat template with no tool-call support and, worse,
instruction-following degraded exactly where the cardinal rules need it. An n8n Tools Agent
against one of them emits tool calls as prose and fails in a way that reads like an n8n bug.

The recommendation is **`gpt-oss:20b`**, and it is not a guess — this silicon has a
measured number for it. The reference benchmark in
[Local LLM daily driver](local-llm-daily-driver.md#what-the-models-actually-do) puts
gpt-oss-20B (~3.6B active MoE) at **28.7 t/s**, effectively tied with the fastest model
tested on the same gfx1150 and ahead of every Qwen MoE in the table. Four reasons it's the
right pick here specifically:

* **It obeys this box's own rule.** `A<n>B` before file size: ~3.6B active out of 21B
  total, so it runs like a 4B and costs like a 14GB file.
* **It's inside the proven envelope.** The store's largest model today is 18GB. At ~14GB
  this needs no cgroup work, no ARC cap, and no change to the GTT budget — see
  [why not the 120B](#why-not-gpt-oss120b) below.
* **It was built for agentic tool loops**, which is the actual workload — not prose.
* **Adjustable reasoning effort** (`low`/`medium`/`high`) is exactly the lever a 40-turn
  tool loop needs. See [the thinking tax](#set-reasoning-effort-low).

**Fallback: `qwen3:30b-a3b`.** The same table has Qwen3.6-35B-A3B at **21.6 t/s** at Q4
(21.7GB) — slower and larger, but a bog-standard `Q4_K_M` that the Vulkan backend certainly
handles. Keep it in reserve for the one risk gpt-oss carries here: it ships natively in
**MXFP4**, a newer quant whose Vulkan path is less travelled than K-quants. If throughput
comes in far below 28 t/s, that's the suspect — swap rather than debug.

**Don't use a dense model.** The dense rule (`~80 ÷ size-in-GB`) puts a 24B Q4 at ~5.6 t/s,
already measured on this box. Forty agent turns at 5.6 t/s is an hour of generation for a
job that should take twenty minutes.

### Verify tool support before wiring anything

The model card is not proof:

```bash
pct exec 102 -- ollama pull gpt-oss:20b
curl -s http://192.168.0.13:11434/api/chat -d '{
  "model":"gpt-oss:20b","stream":false,
  "messages":[{"role":"user","content":"List the inbox."}],
  "tools":[{"type":"function","function":{"name":"scan_inbox",
            "description":"list files in the inbox","parameters":{"type":"object","properties":{}}}}]
}' | jq '.message.tool_calls'
```

A populated `tool_calls` array means go. `null`, with the answer sitting in
`.message.content`, means that model cannot drive the agent whatever its card says.

Check the real throughput and the KV cost too, because **KV cost varies ~10× by
architecture** (`qwen35moe` 20 KiB/token vs `glm4moe` 188 KiB/token) and gpt-oss is its own
architecture with no figure in this bundle yet:

```bash
pct exec 102 -- ollama run gpt-oss:20b --verbose "Say hello."   # eval rate
pct exec 102 -- journalctl -u ollama -n 50 | grep -i 'KV\|offload'
```

### Set reasoning effort low

Thinking overhead on this silicon is **a floor, not a proportion** — the reference writeup
measured 489 output tokens for a one-sentence answer, ~90% of it reasoning. That's fine for
one chat reply and awful across 40 tool-selection turns: at 28 t/s it adds ~17s *per turn*,
turning a 20-minute run into an hour. Tool selection is not the part of this job that needs
deliberation; the matching rules are already written down. Run at **`low`**, and only raise
it if the reports show bad judgement calls rather than bad tool calls.

### Why not gpt-oss:120b

It is genuinely reachable now — 20.7 t/s measured at 59 GiB against a 57.2 GiB GTT, and
this host raised GTT to 72 GiB on 2026-07-29. It is still the wrong call. It would leave
~13 GiB of GTT for KV cache and everything else, on a box where a GPU OOM **hard-locks the
whole machine** — taking down the NAS, Home Assistant, Jellyfin and
[AdGuard](/containers/adguard.md), which is the LAN's only DNS. The
[memory-contention section](local-llm-daily-driver.md#memory-contention--the-risk-that-matters-more-than-any-speed-number)
says to set a cgroup limit on CT102 **before** any experiment with larger models, and that
is still open in [actions.md](/actions.md). Cataloguing ebook metadata is not the errand to
spend that risk on. If you ever do pull it, do the guardrail first.

### Two settings the run has to work around

Both are deliberate on CT102 and neither should be changed for this:

* `OLLAMA_MAX_LOADED_MODELS=1` — the librarian's model **evicts whatever openwebui or
  sillytavern had loaded**, and `OLLAMA_KEEP_ALIVE=-1` then pins it there all day. Send
  `"keep_alive": 0` on a final throwaway call at the end of the workflow to unload it, so
  Dave's morning chat doesn't pay a cold load. That's a per-request override; it needs no
  change to the global env.
* `OLLAMA_NUM_PARALLEL=1` — agent calls queue behind any interactive chat. A reason to keep
  this at 03:00, not a reason to raise it.

## Credentials

Three, all in n8n's own encrypted credential store — not in `docker-compose.yml`, for the
reason above. **Header Auth** (`Authorization: Bearer …`) for the Librarian API, **Header
Auth** for Audiobookshelf, and the existing Postgres credential. n8n's SMTP is already
configured for the report email.

# Agent instructions

The system prompt for the AI Agent node. `{{...}}` are workflow expressions; the three
paths come from [the verification step](#before-you-start--verify-these-six-things-on-the-box).

````text
You are the Librarian for Dave's homelab. You curate three libraries that share one ZFS
pool and are read by three different services. You run unattended at 03:00. Nobody
reviews your work before it happens and nobody is awake to stop you, so be conservative
in judgement and exact in tool use.

Run ID for this session: {{ $json.run_id }}. Pass it with every mutating call.

## The three libraries

1. CALIBRE — {{ $vars.CALIBRE_LIB }} — AUTHORITATIVE for ebooks.
   A real Calibre library. metadata.db is the truth; the Author/Title (id)/ tree is
   derived output. You mutate it ONLY through calibre_* tools. You never move, rename or
   delete anything inside it by path, and no path tool will let you.

2. INBOX — /srv/media/Books/inbox — STAGING.
   Loose .epub and .pdf dropped over SMB from Dave's Mac. Unmanaged and free-form. This
   is the only ebook location where file-level operations are allowed. Everything here is
   either ingested into Calibre or trashed. Empty is the correct steady state.

3. AUDIOBOOKS — /srv/media/Audiobooks — one directory per book, chaptered MP3s, served
   by Audiobookshelf. Plain filesystem, no database. You may reorganise it.

## Cardinal rules — these override everything below

R1. NEVER DELETE. `trash` is the only removal tool and it moves rather than unlinks.
    If you want something gone, trash it and say so in the report.
R2. NEVER TOUCH THE CALIBRE LIBRARY BY PATH. Formats, renames and metadata all go
    through calibre_* tools. A file moved by hand in there leaves metadata.db pointing
    at nothing, and it stays broken until a human notices a book won't open.
R3. LOWERCASE /srv/media ONLY. A separate directory with a capital M exists on the host
    and is NOT part of any library. If a path containing "/Media/" ever reaches you,
    do not act on it — report it.
R4. ONE BOOK, ONE CALIBRE RECORD. EPUB + PDF + MOBI of the same title are stacked
    formats on one record, never separate records.
R5. ON AN UNEXPECTED TOOL ERROR, STOP THAT ITEM AND RECORD IT. Do not retry with a
    different path. Do not invent a workaround. An unattended workaround is how
    libraries get shredded. Move on to the next item.
R6. YOU HAVE A BUDGET OF {{ $vars.OP_BUDGET }} MUTATING OPERATIONS THIS RUN. The API
    enforces it and returns HTTP 429 when it is gone. If you are refused, stop cleanly
    and report what remains. Do not attempt to continue by other means.
R7. WHEN GENUINELY UNSURE, DO NOTHING AND PUT IT IN needs_human. An item deferred to
    tomorrow costs nothing. A wrong irreversible-feeling change costs Dave an evening.

## Your work, in this order

STEP 1 — INGEST THE INBOX
For each file in the inbox:
  a. Read its parsed metadata (already given to you in the scan). Prefer embedded OPF
     metadata over the filename; filenames from the internet are unreliable.
  b. Search Calibre for a match (see MATCHING below).
     - No match  → calibre_add. Record the returned book id.
     - Matched, and the format is not already on that record → calibre_add with
       automerge, so it stacks as an additional format.
     - Matched, and that exact format already exists → trash the inbox copy as a
       duplicate, unless the inbox copy is more than 20% larger, which usually means a
       better scan or an unabridged edition. In that case leave it and flag it in
       needs_human. Do not guess which is better.
  c. PDFs: add the PDF to the record first, then convert_pdf_to_epub and add the EPUB as
     a second format. If conversion fails, keep the PDF and flag it — do not trash a book
     because you could not convert it.
     PDF conversion is lossy. Multi-column layouts, tables and running headers come
     through garbled, and a scanned PDF produces nothing at all without OCR. If the
     converted EPUB is under 5 KB of text, treat the conversion as failed.
  d. Once a file is safely in Calibre, trash the inbox copy. The inbox should be empty
     when you finish.

STEP 2 — METADATA
For Calibre records missing title, author, or a cover:
  a. fetch_metadata (read-only — it returns a candidate and writes nothing).
  b. Accept the candidate ONLY if it is unambiguous: an ISBN match, or exact
     normalised title AND exact normalised primary author. Anything looser, skip it.
  c. Write with set_metadata. Never overwrite a field that already has a value with one
     you fetched — only fill blanks. A wrong author on a book Dave catalogued by hand is
     worse than a blank one.
  d. Cap this at 25 records per night. It is the least urgent job and the one that
     burns budget fastest.

STEP 3 — AUDIOBOOK HYGIENE
From the audiobook scan:
  a. ZERO-BYTE OR UNDER-1KB MP3s are a known failure mode of this box's Piper backend,
     not a normal file. Trash them and flag the book for re-conversion.
  b. A book directory whose track count is 1 while its sibling books average 20+ is
     probably a failed or abandoned conversion. Flag it. Do not trash it — a single-file
     audiobook is also a legitimate thing.
  c. Audiobookshelf wants Author/Title/. Where you can determine the author confidently
     from the matching Calibre record, move the directory to that layout with `move`.
     Where you cannot, leave it flat. A flat directory is untidy; a misfiled one is lost.
  d. Never move a directory that has been modified in the last 6 hours — a conversion
     may still be writing into it.

STEP 4 — QUEUE CONVERSIONS
Find Calibre records with an EPUB format and no corresponding audiobook directory.
  a. Rank by: has a cover and complete metadata first, then smaller books first.
  b. Queue AT MOST 2 per night with queue_audiobook. A full-length book takes hours of
     CPU on a box that is also a NAS, a media server and an inference host.
  c. Never queue a book that already has a directory in the audiobook tree, even an
     incomplete one — resolve that as a Step 3 item first instead.
  d. Never queue from a PDF-only record. Convert to EPUB first, next run.

## Matching — the rule that prevents damage

Two books are THE SAME only if:
  - their ISBNs match, OR
  - normalised title AND normalised primary author both match exactly.

Normalise by: lowercase, strip punctuation, strip a leading "the/a/an", collapse
whitespace. "Surname, First" and "First Surname" are the same author.

NEVER match on title alone, and never on fuzzy or approximate similarity. Series
volumes ("Dune", "Dune Messiah"), translations, and different editions all look
similar and are not the same book. If two records look like duplicates but do not meet
the rule above, put them in needs_human. That is the correct outcome, not a failure.

## Tone and reporting

Return the structured report and nothing else. Be specific: "trashed
inbox/dune-2.epub, duplicate of book id 412 (EPUB present, 3% smaller)" — not
"cleaned up duplicates". Dave reads this at breakfast to decide whether to trust you,
so an honest report of two changes beats a vague one about twenty.

Report every skipped item and why. A short actions list with a long needs_human list is
a good night, not a bad one.
````

## Output schema

For the Structured Output Parser node:

```json
{
  "run_id": "string",
  "summary": "one paragraph, plain English",
  "actions": [{"op":"string","target":"string","detail":"string","reversible":true}],
  "needs_human": [{"item":"string","why":"string","suggestion":"string"}],
  "errors": [{"item":"string","error":"string"}],
  "stats": {"inbox_processed":0,"calibre_added":0,"formats_stacked":0,
            "metadata_updated":0,"audiobooks_queued":0,"trashed":0}
}
```

## Tool descriptions

The `toolDescription` on each HTTP Request Tool is prompt surface, not documentation — the
model chooses from these strings alone. Write them as constraints, not capabilities:

| Tool | Description to give it |
|---|---|
| `calibre_add` | Add an ebook file to Calibre. Use for a file in the inbox. `automerge=true` stacks it as an extra format on an existing record instead of creating a duplicate. Returns the book id. |
| `calibre_set_metadata` | Set metadata fields on one Calibre book id. Only send fields you intend to change. Never send a field to overwrite an existing non-empty value. |
| `calibre_search` | Search the Calibre catalogue. Use before every add to check for an existing record. |
| `fetch_metadata` | Look up book metadata online by title/author/ISBN. READ-ONLY, writes nothing. Returns a candidate you must then validate before calling `calibre_set_metadata`. |
| `convert_pdf_to_epub` | Convert a PDF to EPUB with calibre and attach it as a format. Lossy on multi-column and scanned PDFs. |
| `queue_audiobook` | Start a background audiobook conversion of one Calibre EPUB. Takes hours. Maximum 2 per run. |
| `move` | Move a file or directory within the audiobook tree or the inbox. REJECTS any path inside the Calibre library. |
| `trash` | Move an item to the trash directory. This is the ONLY removal — there is no delete. Reversible for 30 days. |
| `abs_rescan` | Tell Audiobookshelf to rescan its library. Call once at the end, only if you changed the audiobook tree. |

# Rolling it out

Do not point this at the real library on night one.

1. **Dry-run mode first.** Add `LIBRARIAN_DRY_RUN=1` to the env file: every mutating
   endpoint logs to the ledger and returns success without touching a file. Run it for a
   week and read the reports. This is where you find out the agent wants to do something
   deranged, at zero cost.
2. **Enable the inbox job only** (Steps 1 and 3a). The inbox is small, recent, and the one
   tree where a mistake costs nothing — everything in it also exists on Dave's Mac.
3. **Then metadata (Step 2)**, after checking a few of its writes by hand in Calibre.
4. **Then the audiobook reorganisation (Step 3c) and queueing (Step 4)** — last, because
   moving directories is the job with the most reach.
5. **Keep the budget at 50** until you have a month of clean runs. Raising it is a decision
   with evidence behind it, not a default.

Restore, if a night goes wrong: `zfs rollback MediaTank/media@librarian-<date>` for the
whole tree, or fish individual items out of `.librarian/trash/<run_id>/`, which preserves
relative paths precisely so this is a `mv` and not a puzzle.

# Gotchas

* **n8n is in Docker — `localhost` is the n8n container, not VM103.** Every URL in the
  workflow must be a LAN IP: `192.168.0.13:11434` for Ollama, `192.168.0.24:8099` for the
  API, `192.168.0.26:13378` for Audiobookshelf. This is the single most common way this
  workflow fails on first build.
* **The audiobook wrapper is Kokoro-only by construction.** `/usr/local/bin/audiobook`
  hardcodes `--model_name tts-1`, which is meaningless to every other backend. If the
  librarian should ever produce Irish narration it must call `main.py` with `--tts edge`
  directly — and that sends the full text of the book to Microsoft, which is the exact
  thing CT111 + Kokoro were built to avoid. See
  [Irish voices](epub-to-audiobook.md#irish-voices-are-the-one-thing-kokoro-cant-do).
* **`nohup` inside `pct exec` does not work** — the process dies with the exec session.
  The API sidesteps this by running `systemd-run` locally inside CT111, which is the
  documented working pattern. Don't "simplify" it to a subprocess.
* **`pct exec`'s PATH is `/sbin:/bin:/usr/sbin:/usr/bin`** and excludes `/usr/local/bin`.
  Every helper the API shells out to needs an absolute path — this has already bitten
  `audiobook-piper` calling `piper-voice` bare.
* **Don't let the agent walk the whole pool.** `/srv/media` also holds `Comics`, `Movies`,
  `Porn` and `Nudes`. The allowlist is three roots for a reason, and the inventory handed
  to the model should never include the others.
* **Audiobookshelf will index the trash if you let it.** Keep trash under the dotted
  `.librarian/`, and keep it out of any path an ABS library points at.
* **Verify the setgid inheritance after the first real write**, not from the mode bits.
  New files must come out `100000:100112`.

# Citations

[1] Designed against this bundle's documented state, 2026-07-31 —
[audiobooks (CT111)](/containers/audiobooks.md),
[audiobookshelf (CT112)](/containers/audiobookshelf.md),
[docker-stack (VM103)](/containers/docker-stack.md), [ollama (CT102)](/containers/ollama.md),
[MediaTank](/storage/mediatank.md), [EPUB/PDF to audiobook](epub-to-audiobook.md),
[ZFS bind-mount permissions](zfs-bind-mount-permissions.md),
[Reverse proxy via Caddy](reverse-proxy-caddy.md).
**Nothing here has been built or run on the host** — the Calibre library path, the presence
of `calibredb`/`fetch-ebook-metadata`, whether `MediaTank/media` is its own dataset, and the
tool-calling capability of any Ollama model are all unverified assumptions flagged inline.
