# Directory Update Log

## 2026-07-31 (2)
* **Picked the model for the librarian agent — `gpt-oss:20b` — and in the process found a
  second blocker that invalidated part of the design written earlier the same day.**
  Both fixes are in [n8n librarian agent](playbooks/n8n-librarian-agent.md).
  **The model choice came from this bundle's own measurements rather than model-card
  reputation.** The reference 13-model benchmark in
  [Local LLM daily driver](playbooks/local-llm-daily-driver.md#what-the-models-actually-do)
  was run on the same gfx1150 silicon and already contains a figure for gpt-oss-20B:
  **28.7 t/s**, effectively tied with the fastest model tested and ahead of every Qwen MoE
  in the table. At ~3.6B active out of 21B it satisfies the box's own
  `A<n>B`-before-file-size rule, and at ~14GB it sits inside an envelope the store's
  existing 18GB model already proves — so no cgroup work, no ARC cap, no GTT change.
  `qwen3:30b-a3b` (21.6 t/s at Q4) is documented as the fallback for the one real risk:
  gpt-oss ships natively in **MXFP4**, a newer quant whose Vulkan path is less travelled
  than K-quants, so a throughput far below 28 t/s means swap rather than debug.
  **`gpt-oss:120b` was considered and rejected on the record**, because it is now genuinely
  reachable — 20.7 t/s measured at 59 GiB, against the 72 GiB GTT this host set on
  2026-07-29 — and that makes it tempting rather than obviously wrong. It would leave
  ~13 GiB of GTT for KV cache and everything else on a box where a GPU OOM hard-locks the
  NAS, Home Assistant, Jellyfin and [AdGuard](containers/adguard.md), the LAN's only DNS;
  the cgroup limit that risk calls for is still an open item. Cataloguing ebook metadata is
  not the errand to spend that on.
  **The blocker: `OLLAMA_CONTEXT_LENGTH=12400`, set globally on CT102** and noted only in
  passing in the daily-driver playbook. The design as first written fed the agent a full
  `calibre/list` dump — 20k+ tokens for a few hundred books, on top of a ~1,500-token system
  prompt and ~800 of tool schemas. **Ollama truncates silently past the ceiling**, and what
  falls out of a context window first is the top of the prompt: the cardinal rules. That
  produces an autonomous agent holding every mutating tool and none of its constraints, at
  03:00, with nobody watching — a correctness failure, not a performance one. Fixed two
  ways: an n8n **Loop Over Items** around the agent so each iteration sees one book plus
  ≤10 candidate records instead of the catalogue, and a per-request `num_ctx` of 32768
  rather than a change to the global env, which openwebui and sillytavern also read. The
  loop is the better agent design independently — small context improves rule adherence,
  and one bad item now fails one iteration instead of poisoning the run — so per-item
  memory was dropped from the workflow at the same time.
  Also recorded: run at **reasoning effort `low`**, because the thinking overhead measured
  on this silicon is a floor rather than a proportion (489 output tokens for a one-sentence
  answer, ~90% reasoning), which is tolerable once and costs ~17s per turn across a 40-turn
  tool loop; and send a final `"keep_alive": 0` call, since `OLLAMA_KEEP_ALIVE=-1` with
  `OLLAMA_MAX_LOADED_MODELS=1` would otherwise leave the librarian's model pinned all day
  having evicted whatever Dave uses interactively. Both are per-request overrides needing no
  change to CT102's deliberate global settings. P3 in [actions.md](actions.md) updated.

## 2026-07-31 (1)
* **Designed an autonomous n8n librarian agent for the Calibre, inbox and audiobook
  libraries, and wrote it up as a playbook — [n8n librarian agent](playbooks/n8n-librarian-agent.md).
  Nothing was built or run: the host is not reachable from the session that drafted this,
  so six load-bearing facts are flagged as unverified assumptions rather than asserted.
  One blocking dependency was found on paper — no model currently in the Ollama store can
  drive a tools agent.**
  **The constraint that shapes the design: n8n cannot see the library.** It runs in Docker
  on [docker-stack (VM103)](containers/docker-stack.md), and a VM cannot bind-mount a ZFS
  dataset — the same fact that put the audiobook converter in an LXC rather than next to
  n8n in Compose. So n8n gets no filesystem access at all and a small **Librarian API on
  [audiobooks (CT111)](containers/audiobooks.md)** does everything that touches a file.
  CT111 was picked over the alternatives (a CIFS mount into VM103, or n8n SSHing to the
  host to `pct exec`) because it already has calibre 8.5.0, ffmpeg and the *whole*
  `/MediaTank/media` tree bind-mounted, and because it can launch conversions locally with
  `systemd-run` — so nothing gets hypervisor access and no SMB credentials land on VM103.
  Following the in-fleet convention that service-to-service calls go to raw `IP:port`
  ([ollama](containers/ollama.md)), it is deliberately **not** proxied: no new Caddy block,
  no new AdGuard rewrite.
  **Full autonomy was Dave's call; the guardrails were the answer to it.** The concern was
  stated plainly — unattended file operations against a 1.4T library is the mode that can
  quietly destroy it — and the design keeps the autonomy while moving every guard out of
  the prompt and into the API, on the reasoning that a prompt is a request and only code
  actually refuses. Six of them: no delete endpoint exists at all (`trash` moves into a
  dotted `.librarian/trash/<run-id>/`, and a host-side timer is the only thing in the
  system that calls `rm`); a ZFS snapshot at 02:50 ahead of the 03:00 run; a path allowlist
  checked after `realpath` rather than by string prefix, because
  `Audiobooks/../Books/Calibre Library` is exactly the shape of path a confused model
  emits; a **per-run mutation budget** returning `429` past 50, which is the guard that
  bounds damage from failure modes nobody predicted; a `flock` making `calibredb`
  single-writer; and an append-only Postgres ledger.
  **Calibre gets a stricter rule than the other two trees** — `metadata.db` is the truth
  and the `Author/Title (id)/` tree is derived output, so a file moved by hand leaves a
  record pointing at nothing until someone notices a book won't open. The API therefore
  exposes *no* generic file operation against that root; `move` and `trash` reject it
  outright and the only way in is `calibredb`. That asymmetry is also why the inbox stays a
  separate tree.
  **The blocking dependency.** All seven models in `AIVault/ollama-models` are
  RP/creative/uncensored finetunes, a class that routinely ships a chat template with no
  tool-call support — an n8n Tools Agent against one fails in a way that reads as an n8n
  bug. The recommendation applies this bundle's own MoE finding rather than a generic one:
  check for an `A<n>B` suffix before file size (the 26B-A4B MoE measures 24.82 tok/s
  against the 27B dense model's 4.28 at nearly identical file size), so `qwen3:30b-a3b`
  first — with an `/api/chat` probe given to prove tool support instead of trusting the
  model card. Filed as a P3 in [actions.md](actions.md), together with the six
  verification checks and a note that Audiobookshelf's first-run setup and the leftover
  CT111 smoke-test artefacts are prerequisites, not related work.
  Also written down: a staged rollout that starts in dry-run for a week and enables the
  inbox job before anything that moves directories, and the box-specific traps the workflow
  will otherwise hit — `localhost` meaning the n8n container rather than VM103, the
  Kokoro-only `audiobook` wrapper, `nohup`-in-`pct-exec` not working, and the setgid
  inheritance that has to be verified after the first real write rather than read off the
  mode bits. Playbook added to [playbooks/index.md](playbooks/index.md).

## 2026-07-30 (1)
* **Executed the iGPU optimisation plan overnight (evening of 2026-07-29 into the early
  hours) — every transferable lever from the Strix Point writeup tested against this host,
  everything measured, only what helped kept. Two levers rejected on their numbers, two
  closed as already-optimal or not-actionable, one capacity raise applied, one model
  swapped on Dave's call — and one self-inflicted outage: the reboot's shutdown loop took
  down LAN DNS and severed the session driving it. Of the optimisation plan itself, only
  the UMA BIOS trip remains open (the AIVault pool-alert P2 — a Phase-1 follow-up, not an
  optimisation lever — also still stands).**
  The plan: the community writeup cited as [3] in the playbook benchmarks the same silicon
  (Ryzen AI 9 HX 370 / Radeon 890M / gfx1150) on a different stack (Unraid + llama.cpp),
  so its levers are hypotheses here, not conclusions. Each one got an A/B with the
  **decision rule fixed before the measurement**, every figure was read from the live
  process or live sysfs rather than config files, and every run is preserved by label in
  `/root/bench-results.csv` on the host. The harness came first: `/root/bench-ollama.sh`
  (fixed prompt, three runs per model, `eval rate` from `--verbose`), hardened after
  review with a 30-minute timeout writing a `TIMEOUT` sentinel, pre-flight checks and
  argument validation.
  **Closed without a change — CPU governor.** The writeup's top lever is moot here: all 24
  threads already run `performance`/EPP-performance under `amd-pstate-epp`, and this board
  exposes no ACPI platform-profile at all. Already optimal, recorded as such.
  **The baseline got real.** Full three-run baseline across all seven models (label
  `baseline-uma32-gtt31-ctx16k-kvq8`), spreads ≤2.9%: gemma-4-26B-A4B (MoE) 24.82,
  Qwen3.5-4B 23.54, Melody1437-26B-A4B (MoE) 19.71, Moonlit-Mirage-12B 10.88,
  MistralRP-7B 10.43 — its first-ever measurement, and 7.7 GB × 10.43 ≈ 80 lands the
  dense 80÷GB rule again — Qwen3.6-27B 4.28, Dolphin-24B Q8 3.26.
  **Rejected — GPU dpm level.** Forcing `power_dpm_force_performance_level` to `high`
  measured **+0.56%** (gemma MoE) and **+0.92%** (12B dense) against a both->3% rule.
  Reverted to `auto`, read-back confirmed. The sub-1% deltas on both model classes
  corroborate the bandwidth-bound diagnosis: clock pinning buys nothing here.
  **Rejected — KV cache f16.** Against q8_0: **−0.20%** (gemma, a slight regression) and
  **+1.10%** (12B — inside its own 1.2% run-to-run spread, i.e. noise). The "f16 is ~10%
  faster" folklore is contradicted here exactly as it was in [3]'s own A/B; q8_0 stays,
  valued as the pure memory lever it is. Reverted with the live process verified both
  directions.
  **No-op by rule — software currency.** Ollama 0.31.1 vs latest v0.32.5 is one minor
  version behind a ≥2-minors threshold: skip. Mesa 25.0.7 is the newest
  bookworm-backports offers, with nothing ≥25.3 available: skip, revisit condition
  documented (backports shipping ≥25.3, or a CT102 rebuild on trixie). [3]'s +19.8%
  prefill from Mesa 25.3+ does not justify pulling toolchain libraries across a Debian
  release boundary into a working Vulkan chain.
  **Kept — GTT raised 31.2 → 72 GiB** (`ttm.pages_limit=18874368
  ttm.page_pool_size=18874368` via GRUB, backup at `/root/grub.bak-20260729`), taking the
  iGPU's addressable pool from 63.2 to **104 GiB** (`total="104.0 GiB"` in Ollama's own
  journal). This needed a full-fleet reboot, and the first attempt caused the session's
  one genuine incident: **the guest-shutdown loop took down AdGuard (CT109) — the LAN's
  only DNS — and severed the driving session mid-task**, because shutting everything down
  includes shutting down name resolution for whatever is orchestrating the shutdown. The
  reboot completed on a second attempt with an IP-only watcher. The lesson is now written
  down twice: in the playbook's UMA/GTT section and as a standing bullet in
  [troubleshooting-gotchas](playbooks/troubleshooting-gotchas.md) — sequence AdGuard last
  down / first up, and expect the outage. Post-reboot, the full 7-model `gtt72` benchmark
  came back flat (worst −2.1%, on Melody, inside its own 2.2% spread) — GTT kept per the
  pre-fixed rule, since its value is capacity, not speed; a >3% drop anywhere would have
  flagged the reboot itself.
  **Model mix — Dave's calls, executed.** Dolphin-24B pulled at Q4_K_M (14 GB) and
  measured against the Q8 on identical post-`gtt72` config: **5.63 vs 3.22 tok/s, 1.75×**
  (predicted ~2× from the byte ratio; dense-rule product 79). Dave chose to keep the Q4
  and remove the Q8 — done, model store **103G → 79.8G** — and explicitly chose *not* to
  prune the slow dense families, which is recorded as a decision rather than left
  dangling. The levers that don't transfer are also recorded so nobody re-litigates them:
  llama.cpp's `-b`/`-ub` sweep and MTP speculative decoding (Ollama exposes neither),
  llama-swap stack specifics, and ROCm (Vulkan is already the right backend here).
  **What remains from the optimisation plan — the sole remaining item:** the UMA half of
  the A/B, dropping the BIOS frame
  buffer 32 → 16GB, which needs Dave physically at the console. (Separately, the AIVault
  pool-usage alert P2 opened in Phase 1 also remains open in [actions.md](actions.md) —
  a monitoring follow-up, not an optimisation lever.) The comparison base is the
  `gtt72` run and the rule is fixed: gemma-4-26B-A4B stays above ~22.0 tok/s (within ~10%
  of 24.49) → keep 16GB. Updated
  [playbooks/gpu-passthrough-ollama-vulkan.md](playbooks/gpu-passthrough-ollama-vulkan.md)
  (tuning notes, UMA/GTT section, the A/B), [containers/ollama.md](containers/ollama.md)
  (pool line, inventory with the Q4/Q8 swap),
  [playbooks/troubleshooting-gotchas.md](playbooks/troubleshooting-gotchas.md) (DNS
  gotcha), and [actions.md](actions.md) (GTT and model-mix P2s closed to Resolved, UMA
  item re-scoped as the sole remaining plan item, AIVault-alert wording refreshed).

## 2026-07-29 (9)
* **Resolved `cl-helper plugin not detected` UI status issue in Character Library.**
  Root cause: `apiRequest` in `app/library.js` automatically prepends `/api`. Passing `/api/plugins/cl-helper/health` to `apiRequest` produced `/api/api/plugins/cl-helper/health` (404 Not Found), causing `checkClHelperPlugin` health checks to fail. Restored `/plugins/cl-helper` endpoint paths for all `apiRequest` invocations across extension modules and restarted `sillytavern.service`. Updated [containers/sillytavern.md](containers/sillytavern.md) and [actions.md](actions.md).

## 2026-07-29 (8)
* **Stripped `accept-encoding` in SillyTavern's `corsProxyMiddleware`.**
  Fixed backend issue in `/opt/sillytavern/app/src/middleware/corsProxy.js` where forwarded `accept-encoding` request headers caused target servers (like CharacterTavern) to return gzipped bodies while `node-fetch` uncompressed the stream, producing garbled gzip binary bytes in client responses. Added `accept-encoding` to `headersToRemove` and restarted `sillytavern.service`. Updated [containers/sillytavern.md](containers/sillytavern.md) and [actions.md](actions.md).

## 2026-07-29 (7)
* **Pre-populated `_proxyOrigins` set in Character Library `provider-utils.js`.**
  Fixed issue where direct browser fetch attempts to `character-tavern.com` returned raw gzipped binary bytes (`\x1f\x8b\x08...`) instead of routing through SillyTavern's `/proxy/`. Pre-loaded third-party provider domains into `_proxyOrigins` so API calls immediately use SillyTavern's CORS proxy. Updated [containers/sillytavern.md](containers/sillytavern.md) and [actions.md](actions.md).

## 2026-07-29 (6)
* **Resolved CharacterTavern JSON.parse unexpected character error in Character Library.**
  Fixed root cause where `CL_HELPER_PLUGIN_BASE` was pointing to `/plugins/cl-helper` instead of `/api/plugins/cl-helper`. Requests to `/plugins/cl-helper/health` returned 404 HTML (`<!DOCTYPE html>...`), which failed JSON parsing. Updated base URLs across extension modules, added `safeJson` error handling, and restarted `sillytavern.service`. Updated [containers/sillytavern.md](containers/sillytavern.md) and [actions.md](actions.md).

## 2026-07-29 (5)
* **Installed cl-helper server plugin for Character Library into CT120 via SSH.**
  Copied `extras/cl-helper` to `/opt/sillytavern/app/plugins/cl-helper`, set ownership to `sillytavern:sillytavern`, verified `enableServerPlugins: true` in `config.yaml`, and restarted `sillytavern.service`. Log output confirmed `[cl-helper] Character Library helper plugin loaded`. Updated [containers/sillytavern.md](containers/sillytavern.md) and [actions.md](actions.md).

## 2026-07-29 (4)
* **Installed SillyTavern-CharacterLibrary extension into CT120 via SSH.**
  Cloned [SillyTavern-CharacterLibrary](https://github.com/Sillyanonymous/SillyTavern-CharacterLibrary) to `/opt/sillytavern/app/public/scripts/extensions/third-party/SillyTavern-CharacterLibrary`, updated directory permissions (`chown -R sillytavern:sillytavern`), and restarted `sillytavern.service`. Verified container status `active (running)`. Updated [containers/sillytavern.md](containers/sillytavern.md) and [actions.md](actions.md).

## 2026-07-29 (3)
* **Documented resolution for SillyTavern CORS proxy error when searching external character hubs.**
  Character searches from hubs like CharacterTavern, PygmalionAI, and WyvernAI fail with `"Search failed: CORS proxy is disabled. Set enableCorsProxy: true in SillyTavern's config.yaml and restart the server"`. The Node.js backend proxy must be enabled to proxy requests around browser CORS restrictions. Setting `enableCorsProxy: true` in `/opt/sillytavern/app/config.yaml` and executing `systemctl restart sillytavern.service` resolves the failure. Updated [containers/sillytavern.md](containers/sillytavern.md) and [actions.md](actions.md).

## 2026-07-29 (2)
* **Dave pointed out that Phases 0–2 produced no throughput improvement. Chasing that down
  found the actual lever — and caught me getting the same class of thing wrong twice, in
  opposite directions, before evidence settled it.**
  The observation was correct and always going to be: Phase 0 was measurement only, Phase 1
  reclaimed disk, and Phase 2's own plan text predicted "this should not change
  single-stream tok/s". Worth stating plainly rather than letting "efficiency plan" imply
  speed that was never on the table.
  **Wrong turn one.** Reasoning that inference is bandwidth-bound, I predicted the 25 GB
  `Dolphin-Mistral-24B` **Q8_0** would run at ~3.3 tok/s. It measured **3.26** — matching
  the bundle's old `~3.2 tok/s` figure almost exactly, so I concluded that figure had been
  **correct all along** and that entry (1)'s "wrong by ~3.4×" claim was the error.
  **Wrong turn two.** Dave held the correction, noting the likely source discussed **Q4**
  models rather than Q8. That undercuts the match: a Q4 at ~3.2 tok/s implies roughly a 45B
  dense model, not the 24B. **Both of my readings inferred a model from one matching
  number.** The defensible statement is narrow: 3.2 tok/s implies **~25 GB of dense
  weights**, satisfied by a 24B at Q8 or a ~45B at Q4, and nothing on record distinguishes
  them. Now recorded as "unattributed, implies ~25 GB dense", with both wrong turns left
  visible — an unqualified measurement can be wrongly *dismissed* as easily as wrongly
  *trusted*, and this one managed both inside a few hours.
  **Then Dave supplied a benchmark writeup covering the same silicon** (Ryzen AI 9 HX 370 /
  Radeon 890M / gfx1150 / 96GB DDR5-5600, but on Unraid + llama.cpp rather than this host's
  Proxmox + Ollama, 13 models). Its central claim — **speed is set by bytes read per token,
  not parameter count** — exposed a real error in what I had just written into the bundle.
  I had generalised `tok/s ≈ 80 ÷ size in GB` as a rule for this host. **It is a dense-only
  rule.** Every model I had benchmarked happened to be dense, which is precisely why it
  looked validated. Tested it directly against `gemma-4-26B-A4B` (16 GB, MoE, ~4B active):
  the rule predicts 5 tok/s, it measured **24.77** — off by 5×.
  **The finding that actually answers Dave's question.** Six models now measured on
  identical hardware and config, spanning **3.26 → 24.77 tok/s (7.6×)**:
  the 16 GB MoE gemma at **24.77**, the 2.7 GB dense 4B at 23.88, the 17 GB MoE
  `Melody1437-26B-A4B` at **19.73**, the 7.5 GB dense 12B at 10.94, the 18 GB **dense**
  `Qwen3.6-27B` at **4.28**, and the 25 GB dense 24B Q8 at 3.26. **The fastest model in the
  store is also one of the largest**, and it beats a nearly identically-sized dense model by
  **5.8×**. For dense models `80 ÷ GB` holds to a few percent (products: 64 / 82 / 77 / 81);
  for MoE it is meaningless. Total size decides whether it fits; **active** parameters
  decide how fast it runs — check the name for an `A<n>B` suffix before the file size.
  **Second finding, from checking the writeup's claims against live state: GTT has never
  been touched here.** `/proc/cmdline` carries no `ttm.*` parameters, so the borrowable
  pool sits at the kernel default — `pages_limit` 8180431 pages = **31.2 GiB**, confirmed
  against `mem_info_gtt_total`. The iGPU therefore addresses 32 GB UMA + 31.2 GiB GTT =
  **63.2 GiB**, which is exactly what Ollama reports and which also corrects a stale
  `48.0 GiB` win-condition in the playbook. This **inverts the guidance the playbook has
  been carrying**: UMA is permanently stolen, GTT is borrowed and returned, and the standing
  advice for this chip is *small UMA, large GTT* — the opposite of this host. So the Phase 3
  A/B has been re-scoped from "32 vs 24GB UMA" (8GB, needs BIOS) to "16GB UMA + raised
  `ttm.pages_limit`" (16GB returned, and **the GTT half needs only a reboot, not console
  access**). Also confirmed this host is *not* affected by Ollama bug #16462 — it reports
  the pool correctly rather than 2.0 GiB.
  Renamed the playbook's "The real performance lever: BIOS UMA frame buffer" — wrong twice
  over — and rewrote that section to cover both pools. Updated
  [playbooks/gpu-passthrough-ollama-vulkan.md](playbooks/gpu-passthrough-ollama-vulkan.md),
  [containers/ollama.md](containers/ollama.md) (inventory re-sorted by measured speed, with
  dense/MoE marked), and [actions.md](actions.md) (Resolved entry corrected again, UMA item
  re-scoped, two new P2s for GTT and for the model mix). The writeup is cited as a
  cross-reference, explicitly **not** as a description of this host — different OS and
  different inference stack, though its gemma-4-26B-A4B at 29.5 tok/s against 24.77 here
  puts them in the same regime.

## 2026-07-29 (1)
* **Executed the local-AI efficiency plan — measure first, then act. Phases 0–2 are done;
  Phase 3 needs Dave at the BIOS. The headline is that the bundle's single inference
  measurement was not just unqualified, it was wrong by ~3.4×.**
  The plan existed because two P2 decisions ([UMA frame buffer](actions.md), [AIVault's
  203G reservation](actions.md)) had been open since 2026-07-25/26 and neither could be
  ranked honestly against a bundle containing exactly one inference number — `~3.2 tok/s`,
  with no model, quant, context or date attached.
  **Phase 0 — baseline.** Inventoried the model store for the first time: **seven models,
  ~94 GB nominal / 89.7G on disk**, no orphaned blobs, nothing to reclaim. The plan had
  suspected ~80G of unexplained bulk; there is none, which retires that concern rather than
  confirming it. Worth noting for later: four of the seven (76 GB) sit well above the
  7–14B q4 band the playbook calls "the pleasant range" here, and one 24B is a Q8.
  Then built a repeatable benchmark — fixed prompt, three runs per model, reading
  `eval rate` from `--verbose`. It is **very** repeatable: `Moonlit-Mirage-12B` returned
  10.94 / 10.93 / 10.94 tok/s (0.1% spread), `Qwen3.5-4B` 23.88 / 23.34 / 23.84 (2.3%).
  **The `~3.2 tok/s` figure reproduces nowhere near** — the real 12B number is ~3.4× higher.
  That number had been load-bearing for the UMA decision, so it is now retired from the
  playbook outright rather than merely annotated, and anything previously reasoned from
  "iGPU inference here is ~3 tok/s" is worth revisiting. One trap recorded for whoever
  re-runs this: the 4B ignores the word limit and rambles, so its `eval count` swung
  1810 → 8094 → 2853 tokens between runs — **`eval rate` is stable, wall-clock is not**.
  **Phase 1 — reclaimed the AIVault reservation.** `zfs set refreservation=none
  AIVault/vm-103-disk-0` took the pool from **164G to 368G free** in one command. This was
  a fourth option [actions.md](actions.md) had not listed, and it beat both options that
  would have reclaimed space: shrinking the volume means `resize2fs` against a live VM
  disk's ext4 *before* `zfs set volsize`, where reversing the order truncates the
  filesystem; and `sparse 1` governs only future disk creation, doing nothing about 203G
  already reserved. Clearing the reservation needs no filesystem operation, no downtime,
  and reverses in one command. Verified with the same three checks as the original
  2026-07-25 migration — Qdrant `200` on 6333, n8n `200` on 5678, Postgres accepting TCP on
  5432, VM103 running — captured both before and after so the comparison meant something.
  The trade-off was taken knowingly: the reservation was what guaranteed docker-stack's
  disk couldn't be starved, so that became a monitoring concern and a new P2 item for a
  Grafana alert rule. (The plan asserted grafana "has no dashboards built yet" — stale; the
  "N5 Pro Host Overview" dashboard was built 2026-07-25 and already carries a pool usage
  panel, so only the alert rule is missing.)
  **Phase 2 — and here the plan's premise turned out to be simply false.** It called for
  setting `OLLAMA_MAX_LOADED_MODELS` and `OLLAMA_NUM_PARALLEL`, which it said were "not set
  and run at default". Checking the live process first — per the house rule, and it earned
  its keep — showed **both were already explicitly set**, and that the playbook's record of
  the override block was stale in three separate ways: 8 variables documented against 11
  live, `OLLAMA_CONTEXT_LENGTH=12400` documented against `16384` live, and the three
  concurrency knobs missing from the doc entirely. Live values were `NUM_PARALLEL=1` —
  *tighter* than the 2 the plan proposed, so following the plan verbatim would have
  **loosened** the very thing it set out to bound — and `MAX_LOADED_MODELS=2`. Only the
  second was worth changing: with `KEEP_ALIVE=-1` it let two models sit pinned
  indefinitely, observed live at 8.8 GB + 3.2 GB = 12 GB held with both idle. Set to `1`
  after checking with Dave; `NUM_PARALLEL` deliberately left at 1, since doubling KV
  allocation on a bandwidth-bound iGPU works against the goal. Verified against the live
  process rather than the file (the playbook records a past silent failure where an edit
  never reached the process), and the backup copy was moved out of the `.d` directory so it
  couldn't become the next such trap. Eviction confirmed empirically — loaded a second
  model, watched the first drop out of `ollama ps`. Throughput unchanged at 10.96 / 10.93.
  **Phase 3 — prepared, not executed.** The UMA A/B needs BIOS access and a full host
  reboot, so it stays with Dave. What changed is that it is now answerable: the Baseline
  figures **are** the A-side at 32GB, and the playbook and actions.md both carry the exact
  B-side procedure plus a decision rule fixed in advance — **if the 12B stays above ~10.4
  tok/s at 24GB (<5% cost), keep 24GB**; otherwise close as "measured, keeping 32GB". Also
  struck the false claim that justified two prior deferrals: both the playbook and
  actions.md stated there was "no measured inference benefit over 24GB", when **24GB has
  never been run at all**.
  Two things were deliberately not done. The `zfs destroy` of `AIVault/postgres` and
  `AIVault/models` was verified safe (both empty, no snapshots, no guest references) but
  Dave chose to leave them; and `NUM_PARALLEL` was left alone as above.
  Updated [playbooks/gpu-passthrough-ollama-vulkan.md](playbooks/gpu-passthrough-ollama-vulkan.md)
  (new Baseline section, live override block, corrected UMA claim, A/B procedure, stale
  12400/2GB references fixed), [storage/aivault.md](storage/aivault.md),
  [containers/ollama.md](containers/ollama.md) (model inventory, inference performance,
  concurrency bounds), and [actions.md](actions.md) (four items resolved, the UMA item
  corrected and re-scoped, one new P2 for the Grafana alert, the P4 dataset note updated).

## 2026-07-28 (3)
* **Reset the AdGuard admin password on [CT109](containers/adguard.md), and corrected two
  wrong assumptions on the way there — one of them Dave's, one that would have been mine.**
  Dave had forgotten the password and asked for a reset over SSH. Before touching anything,
  read the live `users:` block: one user, `dave`, bcrypt `$2a$` hash, AdGuard **v0.107.78**.
  **The first wrong assumption was that `adguard.133gsl.ie` wanted a different password.**
  Dave had gotten into `http://192.168.0.20` and found the `.ie` name still prompting, which
  reads like a broken reverse proxy or a second credential. It's neither — checked the
  Caddyfile and there is no `basic_auth` anywhere; `@adguard host adguard.133gsl.ie` proxies
  straight to `192.168.0.20:80`, the same AdGuard. AdGuard's session cookie is **host-scoped**,
  so logging in at the raw IP leaves the other two origins unauthenticated. Being logged in
  at the IP was a surviving *session*, not knowledge of the password — worth separating,
  because it's what made the reset still necessary.
  **The second was the assumption that the UI could do it.** Rather than guess from memory
  or click through Settings, grepped the frontend embedded in the binary: the only password
  strings are install-time and login, plus `forgot_password`. No `change_password`,
  `current_password`, `new_password` or `password_changed` key exists, and
  `/control/profile/update` has no password field. **v0.107.78 has no in-UI password
  change** — the config edit was the only route, so the answer to "can I do this from the
  session I already have" is a straight no.
  **Done without the plaintext ever reaching this side.** Dave ran `htpasswd -nBC 10 dave`
  locally (prompting, so nothing in shell history) and handed over only the hash. Stopped
  the service **first** — AdGuard rewrites its config on shutdown, so patching a live file
  and restarting can silently discard the edit, which the existing rewrites procedure
  doesn't warn about. Backed up to `AdGuardHome.yaml.bak-20260728-pwd` (`bak-20260728` was
  already taken by entry (1) the same day), pulled to the host, patched the single bcrypt
  line in Python with an assertion that **exactly one** line matched — abort otherwise, so
  an unexpected second match couldn't be edited blind — then `yaml.safe_load` and diffed
  before pushing back. Diff was one line; all 18 rewrites intact.
  **`htpasswd`'s `$2y$` prefix was left as-is rather than rewritten to `$2a$`.** Go's bcrypt
  validates the major version only and ignores the minor, so `$2y` was expected to work;
  it was flagged to Dave as the first thing to suspect if login failed, and then verified
  working rather than assumed. Also worth recording: the first post-restart health check
  probed `127.0.0.1` and returned `connection refused` plus `http 000`, which looks like a
  dead service — AdGuard binds to `192.168.0.20` specifically. Re-probed on the real
  address: DNS resolving, `/login.html` 200, `/control/status` 401 unauthenticated. Dave
  confirmed the login end-to-end.
  Written up in [DNS via AdGuard](network/dns-adguard.md#admin-password-reset--and-the-session-trap-that-looks-like-a-proxy-bug)
  with a pointer from [adguard (CT109)](containers/adguard.md); two incidental findings
  filed in [actions.md](actions.md).

## 2026-07-28 (2)
* **Audited every domain reference in the bundle against live config, and found two stale
  claims.** Follow-up to entry (1). Extracted all 17 distinct `.lan` hostnames appearing in
  the docs and checked each against both `dig @192.168.0.20` and the live Caddyfile;
  extracted the live backend map and diffed it against the table in
  [reverse-proxy-caddy.md](playbooks/reverse-proxy-caddy.md). **The hostname/port tables
  were entirely accurate** — including the two correctly documented as unproxied
  (`nas.lan`, Samba only; `tailscale.lan`, no web UI). Two errors elsewhere:
  **[jellyfin (CT101)](containers/jellyfin.md) claimed `jellyfin.lan` resolves to
  `192.168.0.12`.** It had been wrong since 2026-07-25, when the Caddy wiring repointed that
  rewrite at `192.168.0.14` — the page was never updated, and it sat under a heading reading
  "IP — confirmed", which is exactly the kind of false confidence that costs an hour later.
  Corrected, with the guest's own `.12` kept and the distinction made explicit: a DNS answer
  is not a way to look up a guest's IP.
  **[docker-stack (VM103)](containers/docker-stack.md) still described `stack-caddy-1` as
  image `caddy:2`.** Stale as of entry (1) the same day — it's now the locally built
  `caddy-cf:2`. Rewritten with the rebuild command and a warning that recreating the
  container from stock `caddy:2` silently stops every `.ie` certificate renewing.
  **Also added the `.ie` name to each service's access line** — jellyfin, openwebui, ollama,
  sillytavern, grafana, sabnzbd, audiobooks, audiobookshelf, code-server, kokoro — plus a
  general note on [containers/index.md](containers/index.md) explaining that the IPs listed
  there are the guests' own and that the DNS names all resolve to `.14`. Historical and
  verification text was deliberately **left unchanged**: those lines record what was actually
  run at the time, and rewriting them would falsify the record.
  Raw-IP references were reviewed and left alone — they're service-to-service API endpoints
  (SillyTavern→Ollama, CT111→Kokoro) or deliberate connectivity tests, where routing through
  Caddy would only add a hop. The one exception got a note rather than a change:
  [AdGuard](network/dns-adguard.md)'s own UI is documented by IP on purpose, since
  `adguard.lan` is resolved by AdGuard itself and won't work precisely when you need it.

## 2026-07-28 (1)
* **Put `133gsl.ie` on Cloudflare DNS and gave the whole lab publicly-trusted HTTPS, without
  opening a port.** Dave bought the domain from Maxer and asked how to migrate DNS to
  Cloudflare and use it for homelab services. Two constraints shaped everything: Cloudflare
  Registrar **doesn't sell `.ie`**, so registration stays at Maxer and only DNS moves; and
  the `.ie` registry runs a **pre-delegation zonecheck**, so the Cloudflare zone had to be
  live and authoritative *before* the nameserver change, the reverse of the usual order.
  There turned out to be nothing to migrate at all — the domain was freshly registered and
  never delegated, so this was a first-time delegation rather than a migration.
  **Design.** Split-horizon, chosen over a public tunnel: the public Cloudflare zone holds
  **one CAA record and nothing else**, while AdGuard resolves `*.133gsl.ie` →
  `192.168.0.14` for tailnet devices. A **wildcard** cert rather than per-service certs,
  specifically because every public cert is published to Certificate Transparency logs —
  15 individual certs would have published a greppable inventory of the lab
  (`sabnzbd.133gsl.ie`, `proxmox.133gsl.ie`, …). This retires the per-device "trust Caddy's
  internal CA" chore for the `.ie` names.
  **Built.** Rebuilt Caddy as `caddy-cf:2` with `caddy-dns/cloudflare` (stock `caddy:2` has
  no DNS providers compiled in, so DNS-01 was simply unavailable), swapped the image in and
  confirmed it was a clean drop-in on the existing `.lan` config *before* writing any `.ie`
  config; added the 18th AdGuard rewrite — the first wildcard one; pre-added
  `sabnzbd.133gsl.ie` to SABnzbd's `host_whitelist`; appended a `*.133gsl.ie` site block
  with 14 services. Turned **Universal SSL off** on the zone, because Cloudflare silently
  appends CAA records for its own five CA providers whenever a CAA record exists — visible
  in `dig` but hidden from the dashboard, so the intended "only Let's Encrypt" policy was
  quietly a five-CA policy until that was found.
  **Four gotchas, all now in the playbook.** `*` must be **quoted** in `AdGuardHome.yaml`
  or it parses as a YAML alias and crash-loops the service. `ns1.maxer.com` is not
  authoritative for Maxer domains (the real backend is `ns1.fastsecurehost.com`), and its
  silence under `dig +short` reads exactly like an empty zone — check for the `aa` flag
  instead. Cloudflare has **two kinds of API token** with different verify endpoints, and an
  account-owned one reports `Invalid API Token` at `/user/tokens/verify` while being
  perfectly valid; roughly 40 minutes were lost treating a working token as broken, and the
  fix was to stop trusting a generic verify endpoint and exercise the actual calls
  (`GET /zones?name=`, then create and delete a TXT). And `qm guest exec … | grep` doesn't
  filter — the guest's stdout returns as one JSON-escaped string, so the pipe must go
  *inside* the guest command.
  **Verified rather than assumed.** Issuer `Let's Encrypt CN=YE1`, subject `CN=*.133gsl.ie`,
  SAN `DNS:*.133gsl.ie`, valid to 2026-10-26. Nine services returned `200`/`302` over
  `curl` **without `-k`**, so the chain validates against the system trust store. All `.lan`
  names re-tested and unaffected; public DNS confirmed to return nothing for any service
  name, which is the split-horizon boundary holding. New playbook:
  [133gsl.ie on Cloudflare DNS](playbooks/dns-cloudflare-133gsl-ie.md); cross-referenced
  from [reverse-proxy-caddy.md](playbooks/reverse-proxy-caddy.md) and
  [dns-adguard.md](network/dns-adguard.md).
  **Two things surfaced in passing**, both filed in [actions.md](actions.md): SSH key auth
  from Dave's Mac had to be set up mid-session (it reached neither the host nor VM103, which
  blocked automating any of this), and `/opt/stack/docker-compose.yml` on VM103 turned out
  to hold **plaintext credentials in a world-readable file** — including a Google app
  password for Dave's Gmail.

## 2026-07-26 (11)
* **Fixed the Thunderbolt link, which had never worked since the Proxmox install.** Dave
  asked for the Mac-mini↔NAS TB connection to be set up after a previous attempt failed.
  The bundle described the link as working-but-limited; the live state was that it was
  **completely down**, and the docs asserted the opposite of reality on the one detail that
  mattered.
  **Root cause.** `network/thunderbolt-link.md` and
  [troubleshooting-gotchas.md](playbooks/troubleshooting-gotchas.md) both said the interface
  "is renamed by udev to `nic0` and never appears as `thunderbolt0`". Live, `ip -br addr`
  showed `thunderbolt0 DOWN` and **no `nic0` at all**. The rename is not a udev rule — it
  comes from `/usr/local/lib/systemd/network/50-pmx-nic0.link`, written by the Proxmox
  installer on 2026-07-04 with a **`MACAddress=02:a4:d5:51:b2:25`** match. `thunderbolt-net`'s
  MAC is locally administered and tied to the current pairing; the live value is
  `02:9b:2f:75:1f:55`, so the match had gone stale and nothing renamed anything. Two
  downstream pieces then silently applied to a nonexistent interface: the `auto nic0` /
  `10.10.10.2/30` stanza in `/etc/network/interfaces`, and
  `/etc/udev/rules.d/99-tb-net.rules`, which matched `KERNEL=="nic0"` — wrong key regardless,
  since `KERNEL` is the kernel name (`thunderbolt0`), not the renamed one. **This also
  corrects entry (17) of 2026-07-25**, which dismissed the `nic0 not recognized` warning
  from `ifreload -a` as "pre-existing, already-documented Thunderbolt hotplug behavior" —
  it was this bug, visible in plain text and misread. The cable and pairing were never at
  fault: `dmesg` had been logging `thunderbolt 0-2: Apple Inc. Mac14,3` all along.
  **Fix.** Replaced the MAC match with `Driver=thunderbolt-net` (originals kept as
  `.bak-20260726`), and rewrote the udev rule to fire off the same driver property with
  `RUN+="/usr/bin/systemd-run --no-block /usr/sbin/ifup nic0"` — `--no-block` because
  `RUN+=` must not block the udev event. The Mac needed no change at all: Thunderbolt Bridge
  (`bridge0`, `10.10.10.1/255.255.255.252`) was already configured correctly and went
  `status: active` by itself the moment the host's end came up.
  **Verified rather than assumed**, per the standing rule. Ping ~0.5ms / 0% loss; SSH over
  TB works (host key fingerprint checked against the known `192.168.0.10` entry before
  pinning `10.10.10.2` in `known_hosts`, not blindly accepted). Then the part worth the
  effort: `modprobe -r thunderbolt_net; modprobe thunderbolt_net` as a stand-in for a
  replug moved the ifindex 66 → 67, proving a real destroy/recreate, and the interface came
  back as `nic0`, up, already addressed — so the hotplug path genuinely works, which is the
  thing the old MAC-pinned config could never do. Throughput: **311 MB/s** host→Mac over TB
  versus **116 MB/s** over the saturated 1GbE, and ~281 MB/s Mac→host raw TCP; both TB
  numbers are bounded by SSH crypto and `nc` at 90% CPU, not the link. **The old 85KB-stall
  symptom did not reproduce** in either direction with checksum/TSO/GSO/GRO left on.
  **Left deliberately undone.** SMB still goes over Ethernet. The link terminates on the
  host, [nas (CT100)](containers/nas.md) has no interface on it, and the `/30` has no spare
  address — moving SMB onto TB means re-addressing the link and bridging into the guest,
  i.e. re-opening the design Dave previously closed. Logged as a P2 decision with the
  measured upside instead of being done unprompted. Also noted in passing that the
  abandoned NAT-over-TB attempt's dead `vmbr1` had `bridge-ports none` **because `nic0` did
  not exist** — the same bug, which means the old "TB is unreliable" verdict rests on a
  broken foundation. MTU is still 1500 (jumbo needs `sudo` on the Mac) — logged as P4.
  **Incidental**: the P3 "unidentified device on `192.168.0.25`" is Dave's Mac mini —
  `ifconfig en0` on the Mac shows MAC `5c:1b:f4:88:46:08` and that address. Rewritten as a
  DHCP-lease-in-the-static-block item rather than a mystery.
  Updated [network/thunderbolt-link.md](network/thunderbolt-link.md) (rewritten),
  [playbooks/troubleshooting-gotchas.md](playbooks/troubleshooting-gotchas.md),
  [containers/nas.md](containers/nas.md), [CLAUDE.md](CLAUDE.md) (the key-facts line
  asserted the wrong thing), and [actions.md](actions.md) (one item identified and
  rewritten, one new P2, one new P4).

## 2026-07-26 (10)
* **Set [CT113](/containers/code-server.md)'s password and closed the websocket question
  entry (9) left open.** Replaced the deb's install-time random password with a fresh
  20-character `openssl rand` value generated in-container, and tightened
  `config.yaml` from the shipped `0644` to `0600` — world-readable is a pointless default
  even in a container with one user. Old config kept as `.bak-20260726`. The value is
  **deliberately not in this bundle**; it was handed over in the session that set it, and the
  page carries the change one-liner instead.
  The interesting half was verification. Entry (9) could only get as far as `200` on the
  login page, which proves nothing about whether the editor actually renders — code-server
  is entirely websocket-driven once the workbench boots, and a proxy that passes HTML
  happily but drops the upgrade produces exactly the grey-screen-forever failure. With a
  password there's a real session, so the whole chain got exercised: `POST /login` → `302`
  plus a `code-server-session` cookie (which is itself an argon2id token, not the password),
  authenticated `GET /` returning workbench HTML rather than the login page, and **all six
  assets the client pulls returning `200`** — including the 17.5MB `workbench.js` bundle.
  Then the part that actually mattered: a raw websocket handshake against
  `/?reconnectionToken=…` returned **`101 Switching Protocols` with `Server: Caddy`**, and
  the server's first frame decoded as a 13-byte VS Code control message of **type 9
  (`Resume`)** — the message a live VS Code server sends to tell the client to start
  streaming. Ran the same probe direct against `192.168.0.27:8080` and got an identical
  result, which isolates the proxy: plain `reverse_proxy` carries it, no `transport` block,
  matching Audiobookshelf's socket.io and unlike `proxmox.lan`.
  Two dead ends worth recording. The Proxmox host **can't resolve `coder.lan`** — its own
  `/etc/resolv.conf` still points at the router, not AdGuard, so every check from there needs
  `--resolve` or a raw socket with an explicit `Host:` header. And the browser route was
  abandoned on purpose: `https://coder.lan` hits Chrome's cert interstitial (the Caddy CA is
  still untrusted on the Mac — the standing P4), the extension can't script an interstitial,
  and getting past the login form would mean typing the password into a field, which isn't
  something to do on Dave's behalf. So **pixels were never rendered** — everything the
  browser would fetch and the protocol it would speak were exercised directly instead, and
  both the container page and this entry say so rather than implying a visual check happened.
  Reworked [actions.md](/actions.md)'s P4 accordingly: the "set your password" item is
  closed, and what remains is the pair of standing facts behind it — the service runs as
  root in-container, and `cert: false` leaves plain HTTP answering on the LAN.

## 2026-07-26 (9)
* **Built [code-server (CT113)](/containers/code-server.md) at 192.168.0.27 and gave it
  `https://coder.lan`** — VS Code in the browser, the fifteenth guest on the box.
  The request read "install Coder Code Server", which names **two different products from
  the same vendor**: `coder/code-server` (one binary, one VS Code instance over HTTP) and
  `coder/coder` (a multi-user workspace platform needing Postgres and a Docker/K8s
  provisioner). Read as the former, which is what the compound name almost certainly means
  and by far the smaller build — but it's a genuine fork in the road rather than a detail,
  so it's called out at the top of the container page: if the platform was wanted, CT113 is
  the wrong artifact and should be replaced, not extended.
  Built to [fleet policy](/network/ip-addressing.md) from the start rather than fixed
  afterwards, same as CT111 and CT112: static IP, AdGuard DNS, and the IPv6-off sysctl in
  place at provisioning time. **Ran the ping check anyway** even though the ledger already
  listed `.27`–`.32` as free from the sweep earlier the same day — `.27` and `.28` both
  silent, no ARP entry. That page says the ping isn't a formality and `.25` is the standing
  proof; taking a nine-hour-old sweep's word for it is exactly the shortcut it warns about.
  Sized past the recent media containers — 4 cores / 4GB and a **32G** rootfs against
  CT112's 16G — on the reasoning that language servers, builds and `node_modules` are the
  actual workload. No bind mounts; if checkouts should live on a pool that's an `mp0` to add
  later, not a rootfs to grow.
  Skipped upstream's `curl … install.sh | sh`. Unlike the Audiobookshelf case in entry (8),
  **that URL is real** — it just platform-detects and fetches the same `.deb`, so the pipe
  into a root shell buys nothing. Took the `.deb` from GitHub Releases directly and recorded
  its sha256 on the container page. Checked for a published checksum file to verify against
  and **there isn't one** — the release carries packages and tarballs only — so the page says
  plainly that the hash is a record of what was installed, not a verification, and HTTPS to
  github.com was the whole integrity guarantee. Better to write that down than to let a
  hash in a doc imply a check that never happened.
  Two decisions documented as decisions rather than defaults. It **runs as root** via the
  deb's `code-server@root` template unit — defensible in an unprivileged LXC where
  container-root is host UID 100000, and near-necessary for a dev box that gets asked to
  `apt install` toolchains, but it does mean the login password is root-in-CT113. And
  `cert: false` with `bind-addr: 0.0.0.0:8080` leaves plain HTTP answering on the LAN, which
  is what every other service here does given there's no active Proxmox firewall — worth
  stating rather than leaving implied.
  **The password was left as the deb's random default and never read**, same call as CT112's
  first-run setup: a credential isn't something to set on Dave's behalf. Both that and the
  root question are in [actions.md](/actions.md) with the commands to change either.
  Wired `coder.lan` → `192.168.0.27:8080` through
  [Caddy](/playbooks/reverse-proxy-caddy.md) (validated before reloading, reloaded without
  downtime) and added the **17th** [AdGuard rewrite](/network/dns-adguard.md) via the
  pull/parse/diff/push procedure that page mandates — YAML parsed to 17 entries, diff showed
  exactly the three intended lines. Verified end-to-end: `302` direct, `dig coder.lan` →
  `.14`, `https://coder.lan/login` → `200` under real DNS, neighbouring rewrites unaffected,
  and a `pct reboot` proving the unit and the IPv6-off state both come back.
  **What wasn't verified is stated as such**: the websocket upgrade the IDE depends on needs
  an authenticated session, which needs the password that was deliberately left alone. A
  `200` on the login page is not proof the editor loads. Recorded on both the container page
  and the playbook, with the likely fix if it turns out to need one.
  One incidental finding: `opencode.lan` was documented as "not proxied — nothing listens",
  but the Caddyfile has always had a block for it and AdGuard a rewrite; only the backend is
  missing, so the domain resolves and *then* fails. Corrected the playbook and filed it P4.
  Also noted throughout: CT113 is the one guest whose DNS name (`coder.lan`) doesn't match
  its hostname (`code-server`) — requested that way, and flagged in three places so a future
  reader grepping `pct list` for "coder" doesn't think the container is missing.

## 2026-07-26 (8)
* **Built [audiobookshelf (CT112)](/containers/audiobookshelf.md) at 192.168.0.26** — an
  Audiobookshelf server to *play* the library [CT111](/containers/audiobooks.md) *writes*.
  The two are complementary, not duplicates, and share nothing but the folder; the
  near-identical names (`audiobooks.lan` vs `audiobookshelf.lan`) are the only real
  confusion risk, so both pages say so explicitly.
  The request arrived as `lxc launch ubuntu:20.04 … && curl -fsSL
  https://audiobookshelf.org/install.sh | bash`, and **none of that survived contact**.
  `lxc launch` is LXD; this is Proxmox. Ubuntu 20.04 has been EOL since April 2025.
  And the installer URL **does not exist** — `audiobookshelf.org/install.sh` returns the
  site's HTML landing page with **HTTP 200**, so `curl -f` doesn't catch it and `bash`
  would have been fed HTML as root. There is no official piped installer; the supported
  path is the project's apt repo, which is what was used. Checking the URL took one fetch
  and was the single most valuable step of the build.
  Two allocation findings, both from verifying rather than assuming. **`192.168.0.25` is
  occupied** by an unidentified device (MAC `5c:1b:f4:88:46:08`) sitting inside the
  documented static range — the obvious next address after CT111's `.24` would have
  produced a live IP conflict. CT112 took `.26`; the stranger is now a **P3** in
  [actions.md](/actions.md) and a row in the
  [ledger](/network/ip-addressing.md). Separately, `.15` is the one free gap in the static
  block but sits in the historical `.12`/`.14`/`.16` collision zone, so it was left alone
  and documented rather than quietly consumed.
  Departed from upstream's install instructions in one more place: the apt key is scoped
  with `signed-by=` to a keyring in `/usr/share/keyrings/` instead of upstream's
  `/etc/apt/trusted.gpg.d/`, which would trust the project's key for *every* repo on the
  box. The hand-written sources line is flagged on the container page as something a future
  package update could overwrite.
  Scoped the bind mount **narrower than CT111's**: `mp0` is
  `/MediaTank/media/Audiobooks` only, not the whole media tree — a player has no need of
  `Books`/`Comics`/`Movies`/`Porn`. No host `chown` was needed; the directory was already
  `100000:100112 2775` from CT111's work and the default idmap presents it as `0:112`
  inside. The deb runs as **uid 999, not root**, so container-root owning the mount bought
  nothing — read access came from the *other* bits and was verified before being relied on.
  Put the service user in mapped GID 112 anyway, so that enabling Audiobookshelf's
  write-metadata-to-library settings later fails on nothing.
  Wired `audiobookshelf.lan` → `192.168.0.26:13378` through
  [Caddy](/playbooks/reverse-proxy-caddy.md) (plain `reverse_proxy`; its socket.io traffic
  needed no `transport` block) and added the 16th
  [AdGuard rewrite](/network/dns-adguard.md). The AdGuard edit went through the
  pull/validate/push dance that page mandates — and the validation **earned its keep**: the
  first run refused to write because the parsed structure differed by more than the new
  entry. That was the checker being over-strict about list ordering rather than a real
  breakage, but it caught a genuine mismatch between what was intended and what was
  produced, on a file that has already crash-looped AdGuardHome once over indentation.
  Verified end-to-end rather than declared done: HTTP 200 direct, HTTP 200 through Caddy
  under real DNS, existing `jellyfin.lan`/`audiobooks.lan` still resolving, and a
  `pct reboot` proving the service returns with the mount ownership and IPv6-off state
  intact. **First-run setup was deliberately not done** — it creates an admin credential,
  which isn't something to do on Dave's behalf.

## 2026-07-26 (7)
* **Finished and verified the local Piper install on [CT111](containers/audiobooks.md),
  and wrote it down — it had been built earlier the same day but existed nowhere in this
  bundle.** Picked the work up by reconstructing state from the box rather than from notes:
  `piper-tts 1.6.0` (OHF-Voice `piper1-gpl`) in its own venv at `/opt/piper1/venv`, an
  adapter at `/opt/piper/piper`, a `piper-voice` installer and an `audiobook-piper` wrapper
  in `/usr/local/bin`, five voices on disk, and a two-chapter smoke test that had already
  rendered successfully at 10:59. So the install was done; the verification and the
  documentation were not.
  **Tested the one combination nobody had.** The earlier smoke test ran from the CLI, where
  `--piper_noise_scale` / `--piper_noise_w_scale` turn out **not to be flags at all** —
  they exist only in the UI, so on the command line the provider stringifies them into argv
  as the literal `"None"` (which is what the adapter drops). The UI sends real numbers down
  a path that had never been exercised. Drove the provider directly with the UI's own
  slider defaults and got real audio out, 22KB of it — so both argv shapes are now proven,
  not just the one.
  **Found and fixed a live bug while checking it.** `audiobook-piper` called `piper-voice`
  bare to auto-install a missing voice, but `pct exec` runs with
  `PATH=/sbin:/bin:/usr/sbin:/usr/bin` — **`/usr/local/bin` is not on it**. The call would
  have died with "command not found" the first time anyone asked for a voice that wasn't
  already downloaded; it was masked only because every test so far used a pre-installed
  one. Patched to an absolute path (backup kept alongside) and then *proved* it by
  deliberately requesting an uninstalled voice — `en_GB-jenny_dioco-medium` downloaded with
  both its `.onnx` **and** its `.onnx.json`, which is the specific thing the upstream
  downloader gets wrong, and rendered. Test voice and output removed afterwards.
  **Measured it instead of guessing: ~20× realtime** (3,540 characters → 227s of audio in
  11.2s, single stream, CPU). That's roughly **4× Kokoro's** measured 4.5–6×, i.e. a
  10-hour book in ~30 minutes rather than ~2 hours. Recorded as a three-backend comparison
  table in the playbook rather than a bare number, since Edge's advantage is parallelism
  and doesn't compare on the same axis.
  **Re-settled the Irish question now that there's a second local backend to check.**
  Piper offers 38 locales but only `en_GB` and `en_US` in English — no `en_IE`, no `ga` —
  confirmed against the live voice index rather than the UI dropdown that the previous
  entry rightly distrusted. So the [Edge privacy trade-off](actions.md) is unchanged and
  stays open at P3; `en_GB-alba-medium` is Scottish and not a substitute.
  Documented three UI traps that would each have cost an evening: the Piper tab's
  deployment dropdown **defaults to Docker** (CT111 has no Docker, so it fails outright),
  the executable-path textbox is **empty** with no default, and the voice dropdowns will
  cheerfully pick a voice that isn't installed and hand you to the broken downloader.
  Updated [the playbook](playbooks/epub-to-audiobook.md) (new Piper and speed sections, the
  Irish section reworked, a new PATH gotcha), [CT111](containers/audiobooks.md) (the
  three-file layout and the wrapper), and [actions.md](actions.md) — where the real
  follow-up is that **nobody has actually listened to Piper yet**, so the speed advantage
  is unbanked until its voice quality is judged against Kokoro's.

## 2026-07-26 (6)
* **Fixed SSH from Dave's Mac to the host**, closing the P3 item raised one entry earlier
  and restoring the live-state verification this bundle's working style depends on.
  Diagnosed before touching anything: `ssh -v` showed the key **was** being offered and
  the server rejecting it, which ruled out the client, the missing `~/.ssh/config` and the
  absent agent in one shot — the problem was entirely host-side. Went in through the
  Proxmox web console (already authenticated as `root@pam`) and found
  `/root/.ssh/authorized_keys` is a **symlink to `/etc/pve/priv/authorized_keys`** on the
  pmxcfs mount, holding three keys: the host's own `root@pve` RSA and two ED25519 keys
  commented `ward@Davids-Mac-mini`. `sshd -T` confirmed `permitrootlogin yes` and
  `pubkeyauthentication yes`, so no config was at fault.
  **Neither `ward@Davids-Mac-mini` key matches anything on the Mac** — it holds
  `zcmOLcbLbSIw…` (`id_ed25519`) and `RF7vr9mXxdSm…` (`pve-backup`), the host has
  `/ReD2UrmLm7fx4…` and `Ux+rd8C64s7HrOQs4…`. That's the drift: the keys were installed
  under a previous keypair or a rebuilt machine and never refreshed. Backed the file up to
  `/root/authorized_keys.bak-20260726`, then appended the Mac's **`id_ed25519`**
  deliberately — it's the default identity, so plain `ssh root@192.168.0.10` works with no
  config file. `pve-backup` was left unused; it's a keypair nothing references.
  Verified the appended line by **fingerprint**, not by reading base64 back — the key was
  typed through a web terminal, where one mangled character yields a plausible-looking
  line that silently never authenticates. Then proved it end-to-end from the Mac:
  `pveversion` over SSH, and finally the two `pct exec 111` checks that had been blocked
  all session — `edge_tts 7.2.7` present, egress to Microsoft live, and the smoke-test MP3
  on the pool at 72KB owned `100000:100112` with the setgid group inherited.
  **The two stale keys were left in place, not deleted** — a backup job or a second device
  could legitimately hold one, and that's Dave's call, not a unilateral change. Filed as a
  new **P3** item instead: two unaccounted-for credentials currently have root on the only
  host in the lab, and removal is now cheap given key auth from the Mac is confirmed.
  Documented the access path in a new *SSH access* section on
  [n5-pro](host/n5-pro.md) — including that authorized keys live on pmxcfs rather than in
  an ordinary local file, which is the part that would send someone editing the wrong
  path. Updated [actions.md](actions.md) (P3 SSH item → **Resolved**, new stale-keys item).

## 2026-07-26 (5)
* **Wrote down two loose ends from entry (4) that had only been mentioned in
  conversation.** Neither is new information — both are now in
  [actions.md](actions.md) so they survive the session.
  **SSH from Dave's Mac to the host is broken**, filed at **P3** rather than housekeeping
  on purpose. `ssh root@192.168.0.10` refuses both keys on the Mac (`id_ed25519`,
  `pve-backup`, no `~/.ssh/config`, no agent); `pve-backup` existing at all says access
  was configured once and drifted. Not a lockout — the web UI is authenticated and
  fine — but it means the live-state verification this bundle's whole working style
  depends on (`systemctl show`, `ls -ln` a device, `cat` the real config) can't currently
  be done remotely. The Edge check in entry (4) only landed because the audiobook UI
  happened to expose a drivable HTTP API, which won't be true next time.
  **The smoke-test artifact is now recorded alongside the existing `rory-test` one** at
  P4, as a pair rather than a single stray file — `Audiobooks/edge-ie-smoketest/` doubles
  as a reference sample of `en-IE-ConnorNeural` if Dave wants to hear the voice before
  committing a book to it, so it has some value beyond being litter.

## 2026-07-26 (4)
* **Smoke-tested the Edge/Irish path and it works — which also proved the caveat written
  an hour earlier in entry (3) had been false when written.** The two `pct exec` checks
  that entry proposed couldn't be run: SSH from Dave's Mac to `192.168.0.10` refuses both
  `id_ed25519` and `pve-backup`. Rather than leave it unverified, the test went in through
  the front door instead — the audiobook UI's own Gradio API on `192.168.0.24:7860`, which
  *is* reachable. Built a minimal single-chapter EPUB locally, uploaded it via
  `/gradio_api/upload`, fired the Edge tab-select and language-change handlers to move the
  server-side state to `en-IE` (skipping that step gets you a Gradio choice-validation
  error, since it validates the submitted voice against the component's *current* choices),
  then submitted `process_ui_form`. Result: `✅ Converted chapter 1`, 183 characters in
  ~1.4s, landing at `/srv/media/Audiobooks/edge-ie-smoketest/0001_Smoke_Test.mp3`. So
  `edge_tts` is in the venv and [CT111](containers/audiobooks.md) has egress to Microsoft
  — the only two things that could have blocked the route Dave chose.
  **The log then showed a full 30-chapter book had already gone through Edge that
  morning**, finishing 09:33 — so "nothing has been synthesised through it" was wrong, and
  entry (3) is corrected here rather than edited. Two facts worth more than the correction:
  that run used **8 concurrent workers** with chapters landing every few seconds, so the
  earlier guess that Edge would be "network- and rate-limit-bound" versus Kokoro's ~4.5–6×
  realtime was backwards — Edge parallelises where Kokoro is a single CPU stream, and the
  UI's **Worker Count defaults to 1**, which throws that away. And it used the UI's
  *relative* default output path and still landed on the pool, which is the
  `WorkingDirectory` fix from entry (2) proven on a real book instead of in theory.
  Also logged into Proxmox via Chrome at `https://proxmox.lan` at Dave's request — the
  session was already authenticated as `root@pam`, no credentials needed or entered. PVE
  **9.2.2**, node `pve`, 11 LXC + 2 VMs all running, CT111 up 9h41m; AIVault still reads
  **64.2%** full, consistent with the reservation issue standing open at P2.
  Two mistakes worth recording: the polling script re-fired the start trigger, so the
  smoke test ran **twice** (10:13:09 and 10:14:42, same output overwritten, harmless), and
  the test directory `Audiobooks/edge-ie-smoketest/` is left on the pool for Dave to bin.
  Updated [EPUB/PDF to audiobook](playbooks/epub-to-audiobook.md) (the "not yet verified"
  paragraph replaced with a measured *Edge is verified working on this box* section, and
  the speed claim corrected) and [actions.md](actions.md) — the P3 verification item moved
  to **Resolved** the same day it was raised, leaving only the privacy trade-off open.

## 2026-07-26 (3)
* **Ran down where the `en-IE` voices Dave spotted in the audiobook UI actually come
  from, and the answer was "nothing to install".** He asked to have them installed; they
  turned out to be **Microsoft's free unauthenticated Edge cloud voices**, not a model
  file — no download, no API key, usable the moment you pick them. Four exist:
  `en-IE-ConnorNeural` and `en-IE-EmilyNeural` (Hiberno-English), plus `ga-IE-ColmNeural`
  and `ga-IE-OrlaNeural` (actual Gaeilge). Confirmed against Microsoft's own voice list
  (322 voices, those 4 are the IE ones) and then, more usefully, by driving the running
  service's `get_edge_voices_by_language` handler on `192.168.0.24:7860` with `en-IE` and
  watching the Voice dropdown repopulate — the offer is real, not just a list in a repo.
  **The genuinely new fact is the negative one: neither local backend can do Irish at
  all.** Checked both live rather than assuming — [Kokoro](containers/docker-stack.md)'s
  60-odd voices are `af_/am_/bf_/bm_/ef/em/ff/hf/hm/if/im/jf/…` with nothing Irish, and
  the UI's **Piper** tab (which hadn't been documented as existing) offers 38 locales
  whose only English entries are `en_GB` and `en_US`. So Edge isn't a preference here,
  it's the only option, and Dave chose it on that basis.
  Documented the trade rather than burying it: Edge ships the full text of the book to
  Microsoft chunk by chunk, which inverts the reason [CT111](containers/audiobooks.md)
  and Kokoro were built on-box in the first place, and it makes throughput
  network-bound instead of the measured ~4.5–6× realtime. Also caught that
  **`/usr/local/bin/audiobook` can't drive Edge** — it hardcodes `--model_name tts-1`,
  which is meaningless outside Kokoro's OpenAI-compatible endpoint — so the playbook now
  carries an explicit `main.py --tts edge` invocation, with the warning that output is a
  *positional* argument there rather than the wrapper's `OUTROOT`, and getting that wrong
  drops MP3s on the container's 16G rootfs instead of the pool.
  **Deliberately not claimed as working.** The dropdown answered in 0.7ms, i.e. from a
  static constant with no network call, so it proves the UI offers the voice and nothing
  more — whether CT111 has `edge_tts` in its venv and egress to Microsoft is untested,
  because SSH to the host refused both available keys this session. Logged as a new
  **P3 — needs verification** section in [actions.md](actions.md) with the two check
  commands and a single-chapter smoke test to run before committing a full book, rather
  than writing the playbook as if it were proven.
  Updated [EPUB/PDF to audiobook](playbooks/epub-to-audiobook.md) (new *Irish voices*
  section, `edge-tts` tag), [audiobooks (CT111)](containers/audiobooks.md) (the wrapper is
  Kokoro-only by construction), and [actions.md](actions.md).

## 2026-07-26 (2)
* **Exposed a web UI for the audiobook pipeline**, after Dave asked whether one existed.
  It did, and it was already installed — epub_to_audiobook ships a **Gradio UI**
  (`main_ui.py`) that came in as a dependency of the earlier pip install, so this was
  wiring rather than installing. Now `audiobook-ui.service` on
  [CT111](containers/audiobooks.md) (enabled at boot, `0.0.0.0:7860`), reverse-proxied to
  **`https://audiobooks.lan`** via [Caddy](playbooks/reverse-proxy-caddy.md) with a
  matching AdGuard rewrite — 15 rewrites now. Same careful path as before: `caddy
  validate` before reload, and the AdGuard YAML edited on a pulled copy, PyYAML-parsed
  and `diff`ed (exactly 3 lines) before pushing back.
  **Two defaults in the UI would have quietly sent work to the wrong place, and both were
  fixed in the unit file rather than left as documentation.** Its default output path is
  *relative*, so with the natural `WorkingDirectory=/opt/epub_to_audiobook` every
  conversion would have landed on the container's 16G rootfs instead of
  [MediaTank](storage/mediatank.md) — CWD now points at `/srv/media/Audiobooks`, verified
  by watching the UI actually create `logs/EtA_WebUI_*.log` on the pool with group
  `100112` correctly inherited. And the UI documents **`OPENAI_API_BASE`** while the
  `openai` SDK reads **`OPENAI_BASE_URL`**; only the latter had been set, so the UI could
  plausibly have fallen through to the *real, paid* OpenAI API rather than erroring. Both
  are now set, and confirmed live in the running process with `systemctl show` rather
  than trusted from the unit file.
  Also checked, not assumed, that the UI's fixed OpenAI voice list (`alloy`, `nova`, …)
  works against Kokoro at all — it does, Kokoro maps the names internally, verified by
  generating real audio with `voice=alloy`. Remaining UI gotchas documented rather than
  worked around, since they're per-conversion choices: the TTS tab defaults to **Edge**
  not OpenAI, the model defaults to `gpt-4o-mini-tts` which fails against Kokoro (must be
  `tts-1`), and upload is **EPUB-only** so PDFs still need the CLI wrapper.
  Updated [audiobooks (CT111)](containers/audiobooks.md),
  [EPUB/PDF to audiobook](playbooks/epub-to-audiobook.md) (new web-UI section plus the
  SMB upload route via the existing `[media]` share — confirmed `dave` is in group 112 and
  can genuinely write, by creating a file as that user rather than reading mode bits),
  [reverse-proxy-caddy](playbooks/reverse-proxy-caddy.md),
  [dns-adguard](network/dns-adguard.md), [mediatank](storage/mediatank.md), and
  [containers/index.md](containers/index.md).

## 2026-07-26 (1)
* **Built an EPUB/PDF → audiobook pipeline**, split across two guests for a specific
  reason: Kokoro is a network service that never touches the filesystem, while
  epub_to_audiobook is a batch CLI that needs direct [MediaTank](storage/mediatank.md)
  access — and a VM can't bind-mount a ZFS dataset. Putting both on docker-stack would
  have meant a CIFS round-trip back through [nas](containers/nas.md) for no gain.
  **Kokoro-FastAPI** (`ghcr.io/remsky/kokoro-fastapi-cpu`, 5.42GB) added as a four-line
  service in docker-stack's existing `/opt/stack/docker-compose.yml` (backed up first),
  serving an OpenAI-compatible TTS API on `192.168.0.14:8880`; brought up with
  `docker compose up -d kokoro` so the other four containers were left alone — verified
  after the fact that caddy and n8n still showed `Up 6 days`. **CPU-only on purpose**:
  Kokoro ships no Vulkan build, its ROCm image is experimental and x86-only against a
  `gfx1150` this homelab already found flaky for ROCm, and the iGPU is already shared
  between ollama and jellyfin. Measured **~4.5–6× realtime**, which makes a 10-hour book
  about a 2-hour job — the GPU wasn't worth the fight.
  **New [audiobooks (CT111)](containers/audiobooks.md)** at `192.168.0.24` (Debian 13,
  4 cores/4GB/16G rootfs) running epub_to_audiobook in a venv, plus calibre 8.5.0 for the
  PDF→EPUB step (epub_to_audiobook takes EPUB only — PDF support is entirely calibre's,
  and is lossy on multi-column or scanned documents). Built to
  [fleet policy](network/ip-addressing.md) from the start — static IP, AdGuard DNS,
  IPv6-off sysctl at provisioning time — rather than drifting and being fixed later like
  most of the fleet was on 2026-07-25. `features: nesting=1` turned out to be required,
  not cosmetic: `pct create` warns that Debian 13's systemd 257 needs it.
  Wrapped the whole thing in `/usr/local/bin/audiobook`, which bakes in the two flags
  that are easy to lose: `--model_name tts-1` (Kokoro fails on the tool's default model
  name) and `--no_prompt` (without it the tool blocks on an interactive `input()` and
  dies `EOFError` under any automation). Added `kokoro.docker.lan` to
  [Caddy](playbooks/reverse-proxy-caddy.md) and AdGuard, validating the Caddyfile with
  `caddy validate` before reloading and parsing the AdGuard YAML with PyYAML before
  restarting — that file has crash-looped the service over indentation before, so the
  edit was done on a pulled copy and `diff`ed (exactly 3 lines) rather than `sed`ed in
  place. Verified end-to-end rather than assuming: `/health` 200, real MP3 out of
  `/v1/audio/speech`, then a full chapter of an actual book from the library rendered to
  **883s of 128kbps audio across 9 chunks with zero errors**, landing on MediaTank as
  `100000:100112` — group correctly inherited via the setgid bit on the new
  `media/Audiobooks` directory, which is the part that would have silently broken
  Jellyfin's access if missed. Also confirmed the four unrelated `.lan` domains still
  resolve correctly after the AdGuard restart.
  Two things surfaced along the way and were **flagged rather than fixed**: AIVault has
  gone from 90.6G to 294G used since yesterday's doc, which turned out to be entirely a
  `refreservation` — `vm-103-disk-0` is thickly provisioned (`sparse 0`) and holds 203G
  against 19.7M of actual data, capping the Ollama model store at 164G; and the earlier
  "nohup inside `pct exec`" trick for long jobs silently doesn't work (the process is
  killed with the exec session — the first provisioning run never ran at all), so
  `systemd-run` is now the documented way to detach.
  New pages: [audiobooks (CT111)](containers/audiobooks.md) and
  [EPUB/PDF to audiobook](playbooks/epub-to-audiobook.md). Updated
  [containers/index.md](containers/index.md),
  [docker-stack](containers/docker-stack.md), [ip-addressing](network/ip-addressing.md),
  [dns-adguard](network/dns-adguard.md),
  [reverse-proxy-caddy](playbooks/reverse-proxy-caddy.md),
  [playbooks/index.md](playbooks/index.md), [mediatank](storage/mediatank.md),
  [aivault](storage/aivault.md), [host/n5-pro.md](host/n5-pro.md),
  [index.md](index.md), and [actions.md](actions.md) (1 new P2, 2 new P4).

## 2026-07-25 (18)
* **Brought the remaining plan decisions back to Dave rather than assuming**, then acted
  on what came back. Four questions, four different outcomes:
  **BIOS UMA (32GB vs 24GB): revisit later**, no BIOS change — noted in actions.md, left
  open.
  **docker-stack's idle 200G AIVault disk: wire it in.** Executed the migration live:
  stopped `postgres`/`qdrant` containers for consistency, wiped the stale LVM signature on
  `virtio1`, `mkfs.ext4` + mounted at `/mnt/aivault`, `rsync -aHAX`'d `/data/postgres` and
  `/data/qdrant` onto it, verified **byte-identical** with `diff -rq` before touching
  anything original, renamed originals aside as a safety net
  (`/data/{postgres,qdrant}.old-20260725` — not yet deleted), then bind-mounted the new
  disk back onto the original `/data/postgres` and `/data/qdrant` paths via `/etc/fstab`.
  `/opt/stack/docker-compose.yml` needed **zero edits** — the bind-mount source paths the
  compose file references never moved, only what backs them did. Restarted both
  containers and verified properly, not just "container started": Postgres's own log
  shows it recognized the existing database ("appears to contain a database; Skipping
  initialization") and reached "ready to accept connections"; Qdrant came up clean on
  6333/6334; n8n (which depends on Postgres, never stopped) answered `200` throughout,
  confirming the dependency chain wasn't disrupted.
  **`MediaTank/Media` vs `media`: investigate before touching.** Checked
  [nas](containers/nas.md)'s `smb.conf` first — only one share exists, `[media]` →
  `/srv/media` (the lowercase, active one). `Media` isn't exposed over Samba at all right
  now, so this was never a live collision, just dormant capacity. Its content (dated
  2026-07-04, untouched since, named like typical downloaded releases) reads as an
  orphaned pre-reorganization landing spot rather than the Mac backup first assumed. Dave
  chose to leave both directories as-is.
  **Reliability posture (AdGuard DNS SPOF, no ZFS redundancy): confirmed accepted
  tradeoffs**, no changes — just wanted to check these weren't oversights before leaving
  them as standing documented facts.
  Updated [storage/aivault.md](storage/aivault.md),
  [containers/docker-stack.md](containers/docker-stack.md),
  [storage/mediatank.md](storage/mediatank.md), and [actions.md](actions.md) (2 items
  resolved, 2 new small P4 cleanup notes for the obsolete AIVault datasets and the
  `.old-20260725` safety-net copies).

## 2026-07-25 (17)
* **Cleared the plan's three "no decision needed" housekeeping items** — all had been
  confirmed dead/inert during earlier audits, so executed directly rather than re-asking:
  (1) Removed the leftover NAT-over-TB plumbing: `pct set 100 -delete net1` plus the
  `vmbr1` stanza dropped from the host's `/etc/network/interfaces` (backed up first to
  `.bak-20260725`), applied live via `ifreload -a`. Verified `vmbr1` no longer exists, no
  `eth1` inside [nas](containers/nas.md), and confirmed main connectivity (`vmbr0`, `.lan`
  DNS resolution) unaffected — the "nic0 not recognized" warning during `ifreload` is the
  pre-existing, already-documented Thunderbolt hotplug behavior, not a regression. (2)
  Deleted the stray `DataPool/subvol-107-disk-0@test-snap` snapshot. (3) Recursively
  destroyed `MediaTank/.system` (6 tiny inert TrueNAS-middleware datasets, all confirmed
  `mountpoint: legacy` i.e. never auto-mounted) — pool root now only holds the real
  content (`Media`, `media`, `backups`).
  Updated [containers/nas.md](containers/nas.md),
  [network/thunderbolt-link.md](network/thunderbolt-link.md),
  [storage/mediatank.md](storage/mediatank.md), and [actions.md](actions.md) (3 items
  moved to Resolved).

## 2026-07-25 (16)
* **Started executing the approved homelab optimisation plan** (`/plan` →
  `/Users/ward/.claude/plans/fluffy-stirring-beacon.md`), beginning with the two
  "ready to execute, no decision needed" items.
  **Reconciling the host RAM figure surfaced a bigger finding than expected.**
  `free -h` confirmed 62Gi total (matching the lower disputed figure) while `dmidecode -t
  memory` confirmed 96GB physically installed (2×48GB DIMMs) — both were right. Root
  cause, found via `dmesg | grep VRAM`: the iGPU's BIOS UMA frame buffer reserves **32GB**
  for itself before the OS ever sees it — not the "2GB" `playbooks/gpu-passthrough-ollama-vulkan.md`
  claimed. That doc's own recommended sweet spot is 24GB; 32GB was already flagged in the
  same doc as "defensible but marginal past 24" *before* anyone had actually checked the
  live value. Corrected the playbook (was stale, not just imprecise) and logged the
  32GB-vs-24GB question as a new P2 decision — reducing it would reclaim 8GB for the rest
  of the fleet, but changing it needs a physical BIOS reboot only Dave can do.
  **Built the first Grafana dashboard** ("N5 Pro Host Overview": host CPU/memory/uptime,
  per-guest CPU/memory, storage pool usage %, sourced from real `pve_*` metric names and
  label formats confirmed live against `pve-exporter` first, not guessed). Hit two real
  gotchas back to back, both stemming from the same root pattern: **this Grafana 13.1.0
  build's actual runtime paths come from `cfg:`/`GF_PATHS_*` overrides on the live
  process, not from `grafana.ini`, and tools invoked without matching those overrides
  silently operate on the wrong location.** (1) Classic file-based dashboard provisioning
  (`/etc/grafana/provisioning/dashboards/*.yaml`) produced zero errors and zero dashboards
  — extensive debugging (permissions, YAML/JSON validity, ownership, AppArmor, provider
  config location) all checked out clean before concluding this build's newer
  `dashboard.grafana.app` unified-storage backend has made the classic provisioner (and
  the `dashboard` SQL table used to verify it) effectively vestigial in this version.
  Switched to the HTTP API (`POST /api/dashboards/db`) instead — worked immediately once
  tried. (2) No known Grafana credentials existed (not the `admin`/`admin` default), so
  `grafana cli admin reset-admin-password` was needed to unblock the API approach — its
  first run silently touched the *wrong* database (defaults to a path under
  `/usr/share/grafana` unless given the same `GF_PATHS_DATA` etc. the systemd service
  uses) and had zero real effect despite reporting success. Fixed by passing the matching
  environment variables. New temporary admin password: `N5proTemp2026xyz` — logged as a
  P4 reminder for Dave to change, not something to do on his behalf.
  Verified the whole chain end-to-end rather than trusting any single success message:
  `GET /api/search` shows the dashboard indexed, and querying the exact panel expression
  through Grafana's own datasource proxy returned live current data.
  Updated [playbooks/gpu-passthrough-ollama-vulkan.md](playbooks/gpu-passthrough-ollama-vulkan.md),
  [containers/grafana.md](containers/grafana.md), and [actions.md](actions.md) (one item
  resolved into an explanation + new P2 decision, one new P4 reminder added).

## 2026-07-25 (15)
* **Network config audit**, at Dave's request following the storage audit. Checked the
  host's own `/etc/network/interfaces`, `/etc/pve/firewall/`, `/etc/pve/datacenter.cfg`,
  `ip route show`, and `tailscale status` against what's documented.
  **Found a second, undocumented bridge — `vmbr1`** (`bridge-ports none`, never attached
  to any physical NIC) — and traced it to an actual user: `grep -rl vmbr1
  /etc/pve/lxc/` found [nas (CT100)](/containers/nas.md), which has a second NIC
  (`net1`, `10.10.10.4/24`) sitting on it. This turned out not to be a new mystery: it's
  the leftover, never-cleaned-up plumbing from the NAT-over-TB experiment that
  `network/thunderbolt-link.md` already documents as "tried and abandoned as unreliable"
  — just the docs never mentioned the dead artifact was still configured. Confirmed
  genuinely dead before writing it up: interface is up (`brctl show vmbr1` lists only
  nas's own veth, no uplink) but essentially silent (290B rx / 11KB tx total, just ARP
  noise). Logged as a safe, low-priority cleanup in actions.md rather than removing it
  unprompted.
  **Second finding**: `/etc/pve/firewall/` is completely empty (no `cluster.fw`,
  no `host.fw`, no per-guest `.fw` files) and `datacenter.cfg` has no firewall enable
  directive — meaning the Proxmox firewall subsystem isn't active anywhere on this host,
  despite four guests (nas, jellyfin, ollama, tailscale) carrying a `firewall=1` flag on
  their `net0`. That flag does nothing without the subsystem enabled. Not a bug — this is
  a common minimal-homelab posture (trust the LAN/router boundary) — but worth stating
  explicitly rather than let the `firewall=1` flags imply protection that isn't there.
  Documented on the host page.
  **Third finding**: `tailscale status` on CT105 shows 5 tailnet members, not the 4
  personal-device count implied by the existing docs — an iPad (`ipad155`) that was never
  listed alongside the Mac Mini, MacBook Pro, and iPhone. Added it.
  **Confirmed clean otherwise**: `/etc/network/interfaces.d/` only holds an empty,
  benign Proxmox SDN placeholder file; routing table is exactly what's expected (default
  route + local subnet via `vmbr0`, no static routes, no VLANs); `vmbr0`/`nic0` configs
  match documented values exactly.
  Updated [containers/nas.md](containers/nas.md),
  [network/thunderbolt-link.md](network/thunderbolt-link.md),
  [host/n5-pro.md](host/n5-pro.md),
  [playbooks/tailscale-subnet-router-exit-node.md](playbooks/tailscale-subnet-router-exit-node.md),
  and [actions.md](actions.md).

## 2026-07-25 (14)
* **Storage audit**, at Dave's request following the fleet-services audit. Checked
  `zpool status` (all three pools), `zfs list -t all` (every dataset and snapshot), and
  `/etc/pve/storage.cfg` against what's documented. Found the biggest thing wrong in this
  whole review so far: **the docs' claim that docker-stack's Postgres/Qdrant data lives on
  AIVault was false.** Traced it properly — `AIVault:vm-103-disk-0` (200G) is attached to
  the VM as `virtio1`, but `lsblk` shows it as a bare `LVM2_member` with no volume group
  ever activated, and `/etc/fstab` has no entry for it; `pvs`/`vgs` inside the guest
  confirm only the root disk's VG exists. The real data (`docker inspect`'s bind mounts:
  `/data/postgres` → `/var/lib/postgresql/data`, `/data/qdrant` → `/qdrant/storage`) sits
  on the **root disk** (`DataPool:vm-103-disk-1`) instead, as plain directories — and it's
  tiny (`du -sh`: 47M and 20K). Two small AIVault datasets (`AIVault/postgres`,
  `AIVault/models`, 96K each) are orphaned leftovers from whatever the original plan was —
  a VM can't consume a host dataset via bind-mount the way an LXC does, so however this was
  set up, it was never actually wired in. Corrected the AIVault, DataPool, and
  docker-stack pages rather than leave the false claim standing, and raised the actual fix
  (wire the disk in vs reclaim it) as a P2 decision rather than doing either unilaterally.
  **Second real finding**: `MediaTank/Media` and `MediaTank/media` are two genuinely
  different directories differing only by case — `Media` (132G, uid 3000, `.DS_Store`,
  `Movies/`, looks Mac-originated) vs `media` (1.28T, uid `100103:100112` matching the
  unprivileged-LXC mapped-UID pattern, `Audio/`/`Books/`/`Comics/` — what containers
  actually read). Flagged as a P2 since SMB is typically case-insensitive on Mac/Windows
  clients even though ZFS is case-sensitive underneath — real collision risk, not touched
  without Dave's input on which one's disposable.
  **Smaller finds, documented as P4 housekeeping, not urgent**: a stray manual snapshot
  (`DataPool/subvol-107-disk-0@test-snap`, opencode's rootfs, Jul 12, not part of any
  backup routine); `MediaTank/.system/*` — inert TrueNAS-middleware datasets left over
  from the original import; all three pools have `zpool upgrade` available but unrun.
  **Confirmed healthy** otherwise: all three pools `ONLINE`, no errors, automatic scrubs
  running clean (last Jul 12), capacity all checks out against the documented drive sizes
  (AIVault 472G/90.6G used, DataPool 928G/30.4G used, MediaTank 5.45T/1.41T used) — though
  all three are **single-disk with no redundancy**, a fact the docs implied but never
  stated outright, now made explicit. Updated
  [storage/aivault.md](storage/aivault.md), [storage/datapool.md](storage/datapool.md),
  [storage/mediatank.md](storage/mediatank.md), [storage/index.md](storage/index.md),
  [containers/docker-stack.md](containers/docker-stack.md), and [actions.md](actions.md).

## 2026-07-25 (13)
* **Fleet-wide audit for anything else undocumented**, at Dave's request following the
  Caddy discovery. Checked every LXC's listening ports (`ss -tlnp`) and running services
  (`systemctl list-units --type=service --state=running`, filtered down to non-boilerplate
  units) against what's documented — all matched except one: **`avahi-daemon` running on
  [sabnzbd (CT108)](containers/sabnzbd.md)**, broadcasting mDNS (`sabnzbd.local`), not
  mentioned anywhere and not present on any other guest. `/etc/avahi/services/` is empty
  (just default hostname announcement, no custom service records) — looks like a side
  effect of however SABnzbd was originally installed, not a deliberate choice. Documented,
  not treated as a bug (nothing indicates it's causing a problem).
  Also explicitly closed two open questions rather than leaving them assumed: confirmed
  `docker ps -a` on docker-stack shows exactly the 4 known containers (caddy, n8n,
  postgres, qdrant) — no stopped or hidden ones; confirmed no other LXC runs Docker at all
  (`systemctl is-active docker` → `inactive` on all 10, docker-stack is genuinely the only
  Docker host); confirmed home-assistant has zero Supervisor add-ons installed (`ha addons
  list` → `addons: []`) — it's core-only, nothing like ESPHome or Node-RED running
  alongside it, closing an open question about what that VM actually manages.
  Updated [containers/sabnzbd.md](containers/sabnzbd.md) and
  [containers/home-assistant.md](containers/home-assistant.md).

## 2026-07-25 (12)
* **Built the `https://jellyfin.lan` reverse-proxy fix Dave asked for**, after diagnosing
  the original report (Dave: "https://jellyfin.lan is not resolving") as actually a missing
  port/protocol, not a DNS failure — `jellyfin.lan` resolved fine, but nothing listened on
  `443` (Jellyfin only serves `8096`). Dave asked to set up a reverse proxy properly rather
  than just tell him to add the port to the URL. Initial decision (asked, not assumed):
  nginx in a new dedicated LXC, HTTPS via an internal CA.
  **Before building it**, checked docker-stack (VM103) for backend ports and found
  something nobody knew was there: `docker ps` showed a running `stack-caddy-1` (`caddy:2`)
  container listening on `80`/`443`. Read its Caddyfile
  (`docker exec stack-caddy-1 cat /etc/caddy/Caddyfile`) — it already had `tls internal`
  blocks reverse-proxying jellyfin, openwebui, ollama, opencode, sabnzbd, adguard, grafana,
  proxmox, n8n, and qdrant, none of it documented anywhere in this bundle. Tested it
  directly (`curl --resolve jellyfin.lan:443:192.168.0.14 https://jellyfin.lan/` → `302`) —
  fully working, just disconnected from DNS. Went back to Dave with this before proceeding
  further, since it invalidated the "new nginx LXC" decision; he chose to use the existing
  Caddy instead of building a redundant second proxy.
  **Wired it up**: added a `sillytavern.lan` block to `/data/caddy/Caddyfile` (the one
  service missing from the existing config, presumably because it predates CT120's
  discovery), reloaded Caddy live (`docker exec stack-caddy-1 caddy reload`, zero
  downtime). Then repointed AdGuard's rewrites for the 9 Caddy-covered domains (ollama,
  jellyfin, openwebui, opencode, sabnzbd, adguard, proxmox, grafana, sillytavern) from
  each service's own IP to `192.168.0.14`, and added a new `qdrant.docker.lan` rewrite
  (Caddy already had the block; AdGuard never had a matching entry). Backed up
  `AdGuardHome.yaml` first.
  **Hit two real bugs applying this**: (1) AdGuardHome crash-looped after the edit
  (`go-yaml load error ... did not find expected key`) — root cause was a 2-space
  indentation mismatch in a manually-inserted YAML block; the file's actual convention is
  4-space `- domain:` / 6-space `answer:`/`enabled:`, not the 2/4 it looks like at a
  glance. Measured precisely with `awk 'match($0,/^ */){print RLENGTH}'` against a known-
  good sibling entry to fix it for real, rather than keep guessing. (2) SABnzbd returned
  `403` for `sabnzbd.lan` through the proxy — its `host_whitelist` setting in
  `sabnzbd.ini` only allowed the bare `sabnzbd` hostname; added `sabnzbd.lan` and
  restarted `sabnzbdplus@root.service`.
  **Verified end-to-end** for all 8 newly-proxied domains: real DNS resolution via
  `dig @192.168.0.20`, then `curl` through the resolved IP — all returned expected
  `200`/`302` codes, including `proxmox.lan` (websocket-sensitive console UI, has its own
  transport block in the Caddyfile already).
  Wrote up the whole subsystem in a new playbook since none of it existed before:
  [playbooks/reverse-proxy-caddy.md](playbooks/reverse-proxy-caddy.md) (full domain table,
  how to add a new domain, both gotchas). Updated
  [containers/docker-stack.md](containers/docker-stack.md) (Caddy wasn't mentioned at all),
  [network/dns-adguard.md](network/dns-adguard.md),
  [playbooks/index.md](playbooks/index.md),
  [playbooks/troubleshooting-gotchas.md](playbooks/troubleshooting-gotchas.md), and
  [actions.md](actions.md) (resolved, plus a new P4 reminder that Dave still needs to trust
  Caddy's CA on his own devices — deliberately left to him, not something to script).

## 2026-07-25 (11)
* **Closed the IPv6 P2 item**, at Dave's request to check it after the DNS audit. Confirmed
  live state first: `ip -6 addr show eth0 scope global` via `pct exec` across all 10 LXCs
  plus `qm guest exec` for the 2 VMs. 9 guests (nas, jellyfin, ollama, docker-stack,
  opencode, sabnzbd, adguard, grafana, sillytavern) had global IPv6; tailscale, openwebui,
  and home-assistant (fixed earlier today) didn't. Root-caused *why* openwebui and
  tailscale were clean: openwebui has `/etc/sysctl.d/99-disable-ipv6.conf`
  (`net.ipv6.conf.{all,default,lo}.disable_ipv6 = 1`), commented "set by
  community-scripts" — it was the only guest actually provisioned with IPv6 disabled at
  creation; tailscale's absence of global IPv6 is unrelated (its own network manager, not
  this sysctl). Unlike the DNS case, there was no conflicting documented policy here — the
  "IPv6 off" intent was consistent everywhere, just never enforced on 9 of 12 guests — but
  it's still a real fleet-wide config change, so raised it as a decision anyway rather than
  assume. Dave chose: disable it fleet-wide. Replicated openwebui's sysctl file verbatim on
  the other 8 LXCs and on docker-stack (VM, via `qm guest exec`). Tested on nas (CT100)
  first: `sysctl -p` applied it **live**, no reboot needed — `ip -6 addr show eth0` went
  empty immediately (different from the DNS/IP fixes, which needed a reboot to take
  effect). Rolled out to the remaining 7 LXCs the same way, then docker-stack. Verified: all
  9 previously-affected guests show zero global IPv6 addresses, and confirmed docker-stack's
  IPv4 (`192.168.0.14/24` on `enp6s18`) is unaffected. Updated
  [network/ip-addressing.md](network/ip-addressing.md),
  [host/n5-pro.md](host/n5-pro.md), [containers/index.md](containers/index.md) (also fixed
  a stale "blank DNS fields" reference there and a stale docker-stack DNS note in
  ip-addressing.md, both superseded by entry (10)'s DNS fix), and [actions.md](actions.md).

## 2026-07-25 (10)
* **Audited the rest of the fleet for the same DHCP/DNS drift found in sillytavern**, at
  Dave's request. IP addressing checked out — `pct config` confirmed all 9 remaining LXCs
  (nas, jellyfin, ollama, tailscale, openwebui, opencode, sabnzbd, adguard, grafana)
  already use static `net0` entries, no DHCP surprises there. DNS was a different story:
  `pct exec <id> -- cat /etc/resolv.conf` on each showed 7 of them (nas, jellyfin, ollama,
  openwebui, opencode, sabnzbd, grafana) pointing at the router (`192.168.0.1`), not
  AdGuard — same symptom as sillytavern, docker-stack, and home-assistant before their
  fixes. But this time it wasn't obvious drift: `network/dns-adguard.md` (written
  2026-07-19, before any of today's fixes) explicitly documents "containers inherit DNS
  from the host... which is what you want — the AdGuard path is for tailnet clients," and
  confirmed the Proxmox host's own `/etc/resolv.conf` is also `192.168.0.1` — so those 7
  actually matched the *original documented design*, while the 3 already "fixed" guests
  had quietly deviated from it without an explicit policy call. Surfaced the conflict to
  Dave instead of guessing either direction; decision: **guests should use AdGuard**,
  making the 3 the new standard and the 7 the ones needing a fix. Applied
  `pct set <id> -nameserver 192.168.0.20` to all 7 — confirmed the config saves
  immediately but does **not** take effect on a running container, only after reboot.
  Asked Dave before rebooting since these are actively-used shared services (NAS/Samba,
  Jellyfin, Ollama, SABnzbd downloads, plus openwebui/opencode/grafana); got the OK to
  reboot all 7 now, did so one at a time via `pct reboot`. Verified after: all 7 show
  `nameserver 192.168.0.20` in `/etc/resolv.conf` and `pct list` shows every guest back to
  `running`. [adguard (CT109)](containers/adguard.md) and
  [tailscale (CT105)](containers/tailscale.md) correctly excluded (self-reference /
  MagicDNS respectively). Rewrote the policy statement in
  [network/dns-adguard.md](network/dns-adguard.md) (was actively wrong as of this fix) and
  updated [network/ip-addressing.md](network/ip-addressing.md) and
  [actions.md](actions.md) to match.

## 2026-07-25 (9)
* **Fixed sillytavern's (CT120) DHCP/DNS drift**, closing the P2 item raised in entry (8).
  Decided to convert to static rather than accept the drift risk, matching the fleet
  convention and the home-assistant precedent. Since sillytavern is an LXC (not a VM),
  networking is host-managed, so the fix was done at the Proxmox level rather than inside
  the guest: `pct set 120 -net0
  name=eth0,bridge=vmbr0,hwaddr=BC:24:11:4A:51:E9,ip=192.168.0.22/24,gw=192.168.0.1,type=veth
  -nameserver 192.168.0.20` (kept the existing MAC and the `.22` address it already held).
  The live hotplug step errored (`ipv4: Address already assigned` — expected, since the
  DHCP lease was still active on the interface at that moment) but `pct config 120`
  confirmed the config saved correctly regardless; `pct reboot 120` applied it cleanly.
  Verified after reboot: `ip -4 addr show eth0` shows the static `.22/24` with no DHCP
  lease markers, `/etc/resolv.conf` now points at AdGuard (`192.168.0.20`, was the router),
  `sillytavern.service` came back `active`, and its web UI answers `200` on port 8000.
  Updated [containers/sillytavern.md](containers/sillytavern.md),
  [network/ip-addressing.md](network/ip-addressing.md), and [actions.md](actions.md).

## 2026-07-25 (8)
* **Filled in the grafana (CT110) and sillytavern (CT120) doc stubs**, closing the P4 item
  from [actions.md](actions.md). Used the Proxmox web UI's node Shell (`https://proxmox.lan:8006`
  → node `pve` → Shell) to run `pct exec` against both containers directly — no SSH needed.
  **grafana**: Grafana 13.1.0 (port 3000) plus a co-located Prometheus 2.42.0 instance;
  confirmed via its own `data_source`/`dashboard` tables (queried with `python3`'s sqlite3
  module, since `sqlite3` CLI isn't installed) that it has exactly one datasource
  (Prometheus, localhost) and zero saved dashboards. Prometheus scrapes itself, its own
  node-exporter, and `pve-exporter.service` (a manually-installed venv at
  `/opt/pve-exporter`, not a Debian package) which proxies the actual N5 Pro host
  (`192.168.0.10`) via a dedicated `prometheus@pve` API user — all three targets healthy.
  **sillytavern**: runs natively via `systemd` (`sillytavern.service`, Node v22.23.1,
  `npm run start`), not Docker — confirmed by `nesting=0` in `pct config 120`. Connects
  directly to [ollama (CT102)](containers/ollama.md) at `192.168.0.13:11434`
  (`hf.co/mradermacher/Moonlit-Mirage-12B-i1-GGUF:latest` for chat, `mxbai-embed-large` for
  embeddings), bypassing [openwebui (CT106)](containers/openwebui.md) entirely. While there,
  **found a new issue**: `pct config 120` shows `net0: ...,ip=dhcp` — sillytavern's
  `192.168.0.22` is a DHCP lease that's simply held steady, not a static assignment like the
  rest of the fleet. Logged as a new P2 in [actions.md](actions.md), same class of risk as
  the home-assistant DHCP issue fixed earlier today. Checked for leftover ZFS/LVM volumes
  in the CT111–119 range to explain the numbering jump to CT120 — found none, so that
  remains unconfirmed. Updated [containers/grafana.md](containers/grafana.md),
  [containers/sillytavern.md](containers/sillytavern.md),
  [network/ip-addressing.md](network/ip-addressing.md),
  [containers/index.md](containers/index.md), and [actions.md](actions.md).

## 2026-07-25 (7)
* **Fixed AdGuard's broken rewrite table** — the P0 item from [actions.md](actions.md).
  7 of 12 domains (`jellyfin.lan`, `openwebui.lan`, `opencode.lan`, `sabnzbd.lan`,
  `adguard.lan`, `proxmox.lan`, `grafana.lan`) were all miscopied to answer
  `192.168.0.14` (docker-stack) instead of their own IPs. Backed up
  `/opt/AdGuardHome/AdGuardHome.yaml` to `.bak-20260725` on
  [adguard (CT109)](containers/adguard.md) first, then via `pct exec 109` ran a `sed`
  with one `-e` per domain, each anchored to that domain's own block
  (`/domain: X\.lan/{n;s/answer:.*/answer: Y/}`) so `n8n.docker.lan` (legitimately
  `.14`) and the already-correct entries couldn't be touched by accident. Restarted
  `AdGuardHome` (`systemctl restart`, came back `active`), then verified all 12 domains
  live with `dig @192.168.0.20` — everything resolves correctly now. Updated
  [DNS via AdGuard](network/dns-adguard.md) and [actions.md](actions.md).

## 2026-07-25 (6)
* **Fixed docker-stack's (VM103) DNS** to point at AdGuard, closing the P3 item from
  [actions.md](actions.md). Patched `/etc/netplan/50-cloud-init.yaml` via
  `qm guest exec 103` (QEMU guest agent, no SSH needed) with a `sed` precise enough to
  hit only the unquoted nameserver line and leave the quoted gateway/route line alone,
  then `netplan apply`. Verified via `resolvectl status enp6s18`: DNS Servers now
  `192.168.0.20`.
* **While verifying, found a much bigger pre-existing bug**: `jellyfin.lan` resolved to
  docker-stack's own IP instead of jellyfin's. Traced it to AdGuard itself — read
  `/opt/AdGuardHome/AdGuardHome.yaml` via `pct exec 109` and found **7 of 12 domain
  rewrites miscopied to `192.168.0.14`** (jellyfin, openwebui, opencode, sabnzbd,
  adguard, proxmox, grafana `.lan`), evidently a copy-paste error. Also discovered the
  Proxmox host's own LAN IP (`192.168.0.10`, via `ip -4 addr show vmbr0`), previously
  undocumented. Logged as a new **P0** in [actions.md](actions.md) — did **not** fix it,
  since it's a bigger change than what was asked (affects 7 tailnet-wide hostnames) and
  needs explicit go-ahead. Updated
  [docker-stack (VM103)](/containers/docker-stack.md),
  [DNS via AdGuard](/network/dns-adguard.md), and [actions.md](actions.md).

## 2026-07-25 (5)
* **Checked docker-stack's (VM103) IP at the source** — the P3 item in
  [actions.md](actions.md). Read `/etc/netplan/50-cloud-init.yaml` via
  `qm guest exec 103` (QEMU guest agent, no SSH needed): `192.168.0.14/24` is hardcoded,
  no `dhcp4`, so it's **confirmed static**, not DHCP fallout — closed that item. Found a
  smaller one in its place: DNS points at the router (192.168.0.1), not AdGuard
  (192.168.0.20), same pattern home-assistant had but lower urgency since the address
  itself doesn't drift. Updated
  [docker-stack (VM103)](/containers/docker-stack.md),
  [IP addressing](/network/ip-addressing.md), and [actions.md](actions.md).

## 2026-07-25 (4)
* **Fixed home-assistant's DHCP/DNS issue** (the P1 item in [actions.md](actions.md)).
  Verified `192.168.0.23` was free (`ping` from the host, 100% loss), then ran
  `ha network update enp6s18 --ipv4-method static --ipv4-address 192.168.0.23/24
  --ipv4-gateway 192.168.0.1 --ipv4-nameserver 192.168.0.20 --ipv6-method disabled` from
  the HAOS CLI via the Proxmox console. Verified after: `network info` shows
  `method: static`, IPv6 `method: disabled`; `.23` reachable and old `.213` lease gone.
  Updated [home-assistant (VM104)](/containers/home-assistant.md),
  [IP addressing](/network/ip-addressing.md), [containers/index.md](containers/index.md),
  and closed the item out in [actions.md](actions.md).

## 2026-07-25 (3)
* **Added [actions.md](actions.md)**: prioritised, standing list of confirmed issues and
  follow-ups from the audit (home-assistant DHCP/DNS fix, IPv6 policy decision, disputed
  RAM figure, unconfirmed docker-stack IP mode, grafana/sillytavern doc stubs, and the
  pre-existing DHCP pool overlap). Linked from [index.md](index.md).

## 2026-07-25 (2)
* **Root-caused the home-assistant DHCP question**: Queried `ha> network info` directly
  in the HAOS console (no login required). Confirmed `enp6s18` is set to `method: auto`
  for both IPv4 and IPv6 — it's a genuine DHCP lease, not a static assignment, and its
  nameserver is the router (192.168.0.1) rather than AdGuard. Updated
  [home-assistant (VM104)](/containers/home-assistant.md) and
  [IP addressing](/network/ip-addressing.md) with the confirmed root cause and the fix
  (switch to static via the HAOS CLI or web UI) — **not yet applied**, left for Dave to
  schedule since it's a live change to the household's smart-home VM.

## 2026-07-25 (1)
* **Live audit against Proxmox UI**: Compared this bundle directly against the running
  Proxmox instance at `proxmox.lan`. Added two previously-undocumented guests found
  running — [grafana (CT110)](/containers/grafana.md) and
  [sillytavern (CT120)](/containers/sillytavern.md) — bringing the total to 12. Resolved
  the [jellyfin (CT101)](/containers/jellyfin.md) IP-verification flag (confirmed
  192.168.0.12). Filled in previously-unknown IPs for
  [docker-stack (VM103)](/containers/docker-stack.md) (192.168.0.14) and
  [home-assistant (VM104)](/containers/home-assistant.md) (192.168.0.213, likely DHCP not
  static — flagged for follow-up). Corrected the "IPv6 off on all guests" claim: most
  guests actually carry live global IPv6 addresses; only tailscale and openwebui match
  the documented policy. See [IP addressing](/network/ip-addressing.md) for details.
  **Not changed**: the documented 96GB host RAM figure — Dave confirmed this is correct
  (with 24GB assigned to the docker-stack VM) despite the Proxmox summary page appearing
  to show ~62GB total; this discrepancy needs separate follow-up and was left untouched.

## 2026-07-24
* **Creation**: Established the bundle from the `n5-pro-homelab` and `okf-documenter`
  Claude Skills — [host overview](/host/n5-pro.md), 10 [container/VM](/containers/index.md)
  concepts, 3 [storage pool](/storage/index.md) concepts, 4 [playbooks](/playbooks/index.md),
  and 3 [network](/network/index.md) concepts.
