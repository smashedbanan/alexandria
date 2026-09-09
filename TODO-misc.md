# TODO (misc)

Open items noticed while getting Alexandria running under Claude Code (2026-09-08).

## Server

- [-] **`raw` record carries no session.** The 2026-09-08 `import_document` session linkage attaches
  the chunks only; the `raw` document record is reachable from them via `extracted_from` but has no
  session edge of its own. Parked 2026-09-08: `contains_session_memory` is declared `IN session OUT fact`,
  so linking `raw` needs a new edge table plus a schema migration, and nothing reads it. Add one if a
  session view ever needs the source document directly.

### `list_sessions` follow-ups (2026-09-09)

- [x] Done 2026-09-09: **`docs/session-memory.md` still describes a stored `memory_count`.** Five
  places, not the two estimated: the schema block still listed the dropped column, the "denormalized
  counter maintained on write" sentence, both `count++` lifecycle comments, the `ended_at` paragraph,
  and the `store_memory` tools row. All now say the count is computed live.
- [-] **No debug UI page for sessions.** `/debug` covers memories, clusters, graph, and maintenance;
  sessions are reachable only through the MCP tools or a direct query. Parked 2026-09-09: `list_sessions`
  covers the "which session was that" case from a client. Add a page if session triage from the browser
  is ever needed — `SessionRepo::list` already returns everything a list view would show.
- [-] **`list_sessions` cannot search summaries.** Filters are `agent_id` / `tag` / `finalized` only;
  finding a session by what its summary says means paging. Parked 2026-09-09: substring `CONTAINS` on
  `summary` is one clause if wanted; semantic search would mean embedding summaries on finalize, which
  is a schema change and a separate task.

### Embedding migration follow-ups (deferred from the 2026-09-08 branch review)

- [-] **The `AlexandriaServer` builder default for `retrieve_min_similarity` is kept equal to
  `RetrieveConfig` by hand** (2026-09-09, both 0.10). No test asserts they match: the config type lives
  in the binary crate and the builder in `alexandria-mcp`, and `main.rs` always overrides the builder
  from config, so the builder value only reaches tests. Add an assertion in `src/config.rs` tests if it
  ever drifts again.
- [x] Done 2026-09-09: **The AGENTS.md retrieval note is stale and contradicts
  `docs/configuration.md`.** Corrected `0.30` to `0.10` and dropped the "keep that ordering"
  prescription. The line no longer restates either threshold — it says both are measured and points
  at `docs/minilm-test-data.md` and `docs/configuration.md`, so this particular drift cannot recur.
  Same sweep found the test count in AGENTS.md at 147 against an actual 151; updated too.
- [-] **`CandleProvider::set_cls_pooling` is public API that exists only for one test** (2026-09-09).
  Integration tests cannot see `cfg(test)` items, so the hook is `pub` behind `#[doc(hidden)]`. A cargo
  feature gate (`test-util`, self dev-dependency) would hide it properly; add one if a second such hook
  appears.
- [-] **Boot could refuse an unlocked corpus instead of warning.** Parked 2026-09-08: needs an escape
  hatch (e.g. `migrate-embeddings --assume-model`) so a pre-lock database can still be stamped, and the
  population is almost certainly nonexistent. Add both together if one ever turns up.

### Retrieval benchmark follow-ups (2026-09-09)

- [ ] **Retrieval degrades as the corpus grows, and nothing tracks it.** The third pass in the
  measurements doc puts the same 12 questions against 143 facts and against today's 743: `mean_rank`
  1.42 -> 2.75, `top1` 9/12 -> 7/12, `mean_gap` +0.148 -> +0.077. `hit_min` and `hit_max` are
  identical across both rows, so the targets score exactly what they always did — the loss is purely
  more facts crowding above them. At 5x the corpus the gap has already halved; nothing says the trend
  is linear, and nothing is watching it. Rerun `alexandria bench-retrieval` at the next significant
  corpus size before concluding anything about the shape of the curve. If the gap keeps closing, the
  levers are a reranker over the top N, hybrid keyword+vector scoring, or a larger model — the
  2026-09-08 passes only ruled larger models out at 143 facts, which is no longer the operating point.
  Fourth point added 2026-09-09 by the threshold-sweep run: 807 facts, `mean_rank` 2.83, one
  per-question rank change against the 743 row. That bounds short-term jitter as far smaller than
  the 143 -> 743 move, but 64 facts is not the "next significant corpus size" this item is asking
  for — it does not narrow the shape of the curve. Fifth point 2026-09-09 while verifying the
  `scored == 0` guard: 830 facts, `mean_rank` still 2.83, `top1` still 7/12, `hit_min` 0.338 —
  unchanged from the 807 row, same caveat, still not the corpus jump this item wants.
- [ ] **The question set only targets facts from the original 143.** All 12 targets predate
  2026-09-08 16:26 UTC, so the 590 facts added since are never a correct answer, only distractors.
  That makes the third pass a clean measurement of "fixed questions against a growing haystack",
  which is what was wanted here, but it is not a measurement of retrieval quality on current
  material — nothing checks that a memory stored last week can be found at all. Add questions
  targeting recent facts before reading the bench as a general quality signal.
- [x] Done 2026-09-09: **`bench-retrieval` is not in `README.md`.** Fixed as diagnosed — the gap
  was the missing CLI surface, not the one command. README now has a `## Command Line` section
  covering the bare invocation, both subcommands and `--help`, matching `USAGE` in `src/main.rs`,
  plus the single-writer caveat. `docs/minilm-test-data.md` added to the Documentation table.
- [-] **`docs/minilm-test-data.md` goes stale silently** (2026-09-09, created with the third pass).
  Its results section is a snapshot of a 743-fact corpus that grows every session, so the numbers
  become wrong-but-plausible rather than obviously wrong — there is no "as of" check, only the date in
  the heading. Nothing regenerates it and nothing compares it to a fresh `bench-retrieval` run. Parked
  because the fix is either a CI job that needs the live corpus (which CI does not have) or a
  discipline that will not hold; the date in the heading is the mitigation. Same shape of risk: the
  headline numbers are repeated in the pointer left at the end of
  `docs/plans/2026-09-08-embedding-model-swap-measurements.md`, so a future rerun has two places to
  update and only one of them is the maintained doc.
- [x] Done 2026-09-09: **`docs/configuration.md` now recommends `bench-retrieval` to anyone
  switching models, but the tool only works on this corpus** (2026-09-09, introduced by that same edit). The "Switching
  models on an existing database" paragraph now says `alexandria bench-retrieval` derives
  `[retrieve] min_similarity` and `[recall] min_similarity` from the new model's output. True here,
  false everywhere else: `QUESTIONS` in `src/bench.rs` hardcodes twelve `fact:` record IDs from this
  install, so on any other database every target is absent. `compute()` skips absent targets by
  design, so the run does not fail — it prints `0/12 questions scored` above a row of `NaN` and `inf`,
  and the floor rule's sanity check compares against an infinite `hit_min`. The `0/12` is a clear
  enough signal to a reader who looks, but the advice in the config doc does not warn them. Took the
  second option: `run()` now bails with `anyhow::ensure!(live_metrics.scored > 0, ...)` before any
  output, so an install without the frozen targets gets an error naming QUESTIONS and the frozen-set
  cause instead of a NaN table and a `floor < inf` sanity check. Exits 1. `docs/configuration.md:78`
  left as written, per the option chosen. Verified both ways against a copy of the live data dir:
  12/12 unchanged on the real corpus, and the guard fires with the target IDs temporarily rewritten
  to absent ones.
- [-] **Prose in `src/config.rs` doc comments duplicates `docs/configuration.md` and nothing checks
  them** (2026-09-09, found while closing the floor-rule item). That item named
  `docs/configuration.md:117` as the one place claiming "the rule gives 0.08 for MiniLM"; the same
  sentence was also in the `min_similarity` doc comment at `src/config.rs:100`, so fixing only what
  the TODO named would have left the stale claim live in the source. The two are written independently
  and drift independently. Not worth a mechanism — the same shape as the AGENTS.md drift, and the same
  mitigation applies: when a doc comment and the config reference would both carry a measured number,
  put it in one and point at it from the other. Grep `src/config.rs` for the value before closing any
  future "stale number in configuration.md" item.
- [-] **The `scored == 0` bail guards the live pass only** (2026-09-09, added with that guard). The
  baseline pass is the oldest `BASELINE_SIZE` of the live corpus, so a live corpus that scores at all
  normally carries the targets into the baseline window too, and a live corpus that scores zero never
  reaches the baseline report. The uncovered case needs a database holding those exact record IDs
  where all twelve are outside the oldest 143 — not reachable from any real corpus this tool runs on.
  Guard the baseline separately only if `BASELINE_SIZE` ever stops meaning "the original install".

- [-] **The `bench-retrieval` corpus copy was taken from a live data dir** (2026-09-09). `README.md:55-57`
  says to stop the server for the `cp` and then run against the copy via `ALEXANDRIA_DATA_DIR`; the
  `scored == 0` verification skipped the stop and copied `~/.local/share/alexandria/data` while the
  server held the writer lock. SurrealKV replayed the WAL and reported 830 facts, and the metrics row
  matched the previous run, so nothing detectably tore — but an LSM tree copied mid-write is not
  guaranteed consistent, and a silently truncated copy would look like a corpus-size data point rather
  than an error. Treat the 830 figure as approximate. Stop the server for any copy whose numbers get
  recorded in `docs/minilm-test-data.md`.

- [-] **The baseline pass reconstructs by size, not identity** (2026-09-09). `BASELINE_SIZE = 143`
  takes the 143 oldest active facts, which is not the same set as the 143 that were active on
  2026-09-08: any of those deleted since drops out and the window reaches forward to replace it. It
  reproduced the recorded table exactly this time, so the drift is currently zero-to-negligible, but
  it grows monotonically with every deletion and there is no way to detect it from inside the tool.
  A timestamp cutoff cannot be substituted — see the next item. If the baseline row ever stops
  reproducing, suspect this before suspecting the metrics.
- [-] **The measurements doc's "08:08 data copy" is local time and misleads** (2026-09-09). It reads
  as a UTC timestamp, but no active fact predates 2026-09-08 12:30 UTC: `08:08` is UTC-4 and is when
  the data dir was created, not when the measurement ran. A cutoff built from it selects nothing,
  which is how the size-based reconstruction above came about. Left as written since the third-pass
  section now explains it; worth remembering that every other bare timestamp in these plan docs is
  local too.
- [ ] **`ALEXANDRIA_AUTO_RECALL_LIMIT` is an untuned lever and is the binding constraint for some
  questions** (2026-09-09, from the threshold sweep). At `T = 0.30` every one of the 12 targets clears
  the threshold on score, yet only 10 are delivered: two rank 8th and 7th, outside the hook's
  `limit = 5`, so no threshold can reach them. For those questions the threshold is irrelevant and the
  limit is the whole story. Nothing has ever measured what `limit` should be — 5 is the value both
  clients inherited unexamined. The sweep already computes `hits_delivered` against `RECALL_LIMIT`, so
  sweeping the limit instead of the threshold is a small change to `src/bench.rs`. Do that before
  touching the threshold again; raising the limit and lowering the threshold trade against each other
  and only one of the two has been measured.
- [-] **The pi extension default now diverges further from upstream** (2026-09-09). Flipping
  `recallMinSimilarity` from `0.58` to `0.35` in `contrib/pi/extensions/alexandria-auto-recall/src/config.ts`
  closes the "pending a change to the extension" note that `docs/configuration.md` carried, but
  `contrib/pi` is fork content synced against cebarks/alexandria and this file is a known merge
  surface (`docs/plans/2026-09-09-upstream-sync.md:31`). The divergence was previously documentation
  only and is now behavioural. Expect a conflict on the next sync, and per the `:theirs` lesson below,
  do not resolve it blanket — upstream has no counterpart for the measurement this value rests on.
- [x] Done 2026-09-09: **The floor rule now yields 0.07 against a 0.10 default** (live corpus; 0.08
  on the baseline). `0.10` still stands — `hit_min` is 0.338, so every candidate floor sits far below
  the weakest true hit. The stale claim that "the rule gives 0.08 for MiniLM" is fixed in both places
  it lived, `docs/configuration.md` and the `min_similarity` doc comment in `src/config.rs`; both now
  say the result depends on the corpus as well as the model and drifts down as the corpus grows.

## Build / toolchain

- [-] **Startup memory doubled during model load (2026-09-08).** `candle.rs` reads the safetensors
  file into a `Vec<u8>` (`from_buffered_safetensors`) instead of mmap, so the workspace can carry
  `unsafe_code = "forbid"`. For MiniLM (~90 MB) the buffer and the built tensors coexist until
  `BertModel::load` returns, then the buffer drops; there is no earlier drop point in safe code.
  Parked 2026-09-08: mmap would not remove the copy either (candle still copies each tensor out of
  the map; the source bytes just become reclaimable page cache), and streaming tensors through the
  `safetensors` reader into a `HashMap` holds the same peak. Revisit only if a much larger model is
  adopted; the escape hatch is a `#[allow(unsafe_code)]` on that one call plus
  `from_mmaped_safetensors`.

- [-] **`.githooks/pre-commit` hard-depends on `just`** (2026-09-09, accepted when the hook was
  collapsed to `just fmt` / `just lint` and the hand-copied `cargo` invocations were deleted). A
  fallback to the literal commands would re-create the duplication the collapse removed, so there is
  none: without `just` on PATH the hook dies with `just: command not found`. Judged safe because the
  only documented install path is `just install-hooks` (justfile:44), which already requires it, and
  the failure is loud rather than silent. Add a fallback only if the hook starts being installed some
  other way. Related: the hook's own `→ just fmt` / `→ just lint` labels can no longer go stale, since
  `just` echoes each recipe body as it runs and so prints the real invocation underneath.

- [-] **`just install-hooks` copies the hook instead of symlinking it** (2026-09-09, noticed while
  rewriting it). `.git/hooks/pre-commit` is a snapshot, so every edit to `.githooks/pre-commit` needs a
  re-run of `just install-hooks` and nothing warns you that the installed copy is stale. Pre-existing,
  not caused by the rewrite. A symlink would fix it but breaks on Windows checkouts without developer
  mode; leave the copy unless staleness actually bites someone.
- [-] **`jj resolve --tool :theirs <file>` resolves every conflicted hunk in the file, not just the one
  you are thinking about** (2026-09-09, learned during the cebarks/alexandria upstream sync). The sync
  plan's rules described one hunk per file ("take upstream's paragraph, append our sentence"), but
  `README.md` and `docs/configuration.md` each had several, so blanket `:theirs` silently deleted
  fork-only content upstream had no counterpart for — the journald size-cap section,
  `RUST_LOG=info,rmcp=warn`, the whole Claude Code MCP-client block, `contrib/claude` cross-references,
  every `batch_size` doc, the `min_similarity = 0.10` floor, the "Switching models" paragraph, the
  `1_Pooling/config.json` clause, and the `alexandria migrate-embeddings` command name. Two restoration
  rounds recovered them. Before accepting any future `:theirs`/`:ours` resolve, diff each file's
  base→ours additions against the result and require a three-way grep count (ours/upstream/current);
  an upstream count of 0 means upstream never had the content, so it cannot have superseded anything
  and the loss is silent, not a merge decision.

## Dependencies

- [-] **`tokenizers` still builds `onig`** (2026-09-08). candle-core 0.11 depends on tokenizers with
  the `onig` feature itself, so our `default-features = false` cannot drop the C build. Goes away
  only if a candle bump drops it; `fancy-regex` is the pure-Rust alternative if it ever becomes ours
  to choose. Rechecked 2026-09-09: 0.11.0 (2026-06-26) is still the newest candle-core release.
- [-] **`tokenizers` is held at 0.22 to match candle-core 0.11** (2026-09-08; was 0.23, which built a
  second copy). Bump the workspace pin together with the next candle bump that moves its own.
- [-] **Transitive "Unchanged" `cargo update` entries are upstream pins, not ours.** `generic-array`
  0.14.7, `i_float`/`i_overlay`/`i_shape`, `matchit` 0.8.4, `pdqselect` 0.1.0 stay put even after
  the direct bumps above; they move when the pulling crate does. Rechecked 2026-09-09: `cargo update
  --dry-run` locks 0 packages and every direct dependency is on its latest stable release. Holders:
  `matchit` 0.8 by `axum` 0.8.9 (no axum 0.9 on crates.io yet; `matchit` 0.9 waits on it), the
  `i_*` set by `geo` 0.32 via `revision` in surrealdb-core, `generic-array` 0.14 by `digest` 0.10
  via `argon2` in surrealdb-core, `pdqselect` dev/build only. The next upstream moves that would
  shift them are surrealdb 3.3 (3.3.0-beta.3 is prerelease, not adopted) and axum 0.9.

## Claude Code integration

- [-] **Extraction keeps the Pi `error-resolution` tag unimplemented** (2026-09-09, remainder of the
  tool-error entry closed the same day). `alexandria-extract.sh` serializes `is_error` tool results as
  `[Tool error]: <tool> <its command or file path, else the first 120 chars of its input JSON> --
  <first 300 chars>` (2026-09-09; other tools keep the JSON form, add a field when one gets noisy),
  joined through `tool_use_id` (a result whose call is not in the chunk keeps the bare form), and leaves pairing
  and root-cause judgement to haiku. Accepted noise: permission denials, worktree-isolation refusals,
  and user rejections still go in. Add the tag filter only if junk memories of that shape appear.
- [-] **Stop-hook extraction makes one haiku call per turn** (2026-09-08, retry dropped). The retry on an
  empty first result rested on one observation (empty, then three memories on the same prompt) and
  doubled the cost of every tactical turn; the extract log showed only the second call failing, on
  the shrunken timeout. If `extracted` volume drops noticeably, restore the loop gated on transcript
  size rather than unconditionally.
- [-] **The entrypoint gate is a denylist read off the 2.1.263 bundle** (2026-09-09, closes the "`sdk-*`
  gate does not cover other entrypoints" item). Off: `sdk-*`, `mcp`, `bench`, `claude-code-github-action`,
  `claude-security` (a guess from the name; nothing in the bundle says what it is), `*_trigger`, and Cowork
  (`local-agent`, `claude-coworker*`, `remote_cowork`; turned off 2026-09-09 after first being left on).
  Deliberately on: the `remote*` / Slack / Teams family, since a human types those prompts (and the remote
  ones cannot reach a local server). An allowlist on `cli` was rejected: it would silently turn memory off in `claude-desktop`
  and `claude-vscode`, and a rename would fail closed. The validator table (26 values) is only in the
  bundle, so a new headless entrypoint lands on by default until someone re-reads it; re-grep
  `$Yt={cli:!0,...}` in `/opt/claude-code/bin/claude` after upgrades that add surfaces.
- [-] **Hook development in a live interactive session pollutes the real database.** Companion to the
  `CLAUDE_CODE_ENTRYPOINT` item: the installed Stop hook extracts from this session's transcript too, so stub
  payloads and probe strings from tests pasted into the conversation become `extracted` memories (a
  "Detach debug probe" memory from 2026-09-08 surfaced in auto-recall today). `test.sh` itself is
  clean (own session id, deletes on exit); the leak is the interactive session around it. Parked
  2026-09-08: nothing in the hook can tell a pasted stub payload from a real conversation, so the
  headless fix does not apply. Accepted mitigation: start the developing session with
  `ALEXANDRIA_AUTO_STORE=off`, or delete by hand afterwards.
- [-] **The marker-prune `find` is duplicated by hand in both hooks** (2026-09-09, closes the "runs only
  from the Stop hook" item). `alexandria-recall.sh` now prunes on every prompt with the same expression as
  `alexandria-extract.sh`; nothing checks they match. Two identical lines beat a sourced helper for now;
  factor one out if a third hook needs it or the expression changes.
- [-] **Cross-session extraction dedup covers only what the prompts recalled** (2026-09-08, replaces
  the "dedups within a session only" item). `alexandria-extract.sh` now adds the recall hook's hits, read
  from the `hook_additional_context` attachment lines in the transcript chunk, to `<already_stored>`.
  Measured first: a post-hoc similarity filter on the candidates does not work on MiniLM, since the
  three real duplicates scored 0.64-0.76 against each other and distinct neighbours 0.63-0.76, so
  haiku has to judge. Not covered: a gotcha that surfaces only from tool output (nothing recalled it);
  sessions with `ALEXANDRIA_AUTO_RECALL=off`; and hits recalled for prompts in an earlier chunk of the
  same session, since only the lines past the marker are scanned (scanning the whole transcript is one
  `tail` argument away but would grow the block without bound on long sessions). If duplicates of that
  shape keep appearing, the next step is one `retrieve_memories` per candidate with the top hits fed to
  a second, smaller haiku call.
- [-] **The extract chunk now ends at the last assistant line** (2026-09-09, closes the "queued
  follow-up prompt lands in the previous turn's chunk" item). A prompt dispatched inside the 1 s flush
  wait sits past that line and is extracted with its own turn. Still one turn early if the next turn's
  first assistant message also lands inside the wait; that needs a first-token latency under 1 s, not
  seen yet. If it shows up, cut at the last assistant line that precedes a `queue-operation` `dequeue`
  line instead (Claude Code writes one when it dispatches a queued prompt).
