# TODO (misc)

Open items noticed while getting Alexandria running under Claude Code (2026-09-08).

## This file

- [ ] **23 of 27 entries are `[-]`; this is a decision journal with four TODOs in it**
  (2026-09-09, from the adversarial review pass). `[-]` means "parked, here is why" and most entries
  are write-ups of finished work — the "Claude Code integration" section is seven entries, all `[-]`,
  most opening "closes the X item". The actionable `[ ]` items are buried among them and nobody
  scanning this file finds them first (counts as of 2026-09-09). Split the rationale into
  `DECISIONS.md` and leave `[ ]` items here, or accept that this is a journal and rename it. Not
  done unilaterally: which entries are rationale and which are deferred work is a judgement call
  per entry.

## Server

- [-] **`raw` record carries no session.** The 2026-09-08 `import_document` session linkage attaches
  the chunks only; the `raw` document record is reachable from them via `extracted_from` but has no
  session edge of its own. Parked 2026-09-08: `contains_session_memory` is declared `IN session OUT fact`,
  so linking `raw` needs a new edge table plus a schema migration, and nothing reads it. Add one if a
  session view ever needs the source document directly.

### `list_sessions` follow-ups (2026-09-09)

- [-] **No debug UI page for sessions.** `/debug` covers memories, clusters, graph, maintenance,
  and query; sessions are reachable only through the MCP tools or a direct query. Parked 2026-09-09:
  `list_sessions` covers the "which session was that" case from a client. Add a page if session
  triage from the browser is ever needed — `SessionRepo::list` already returns everything a list
  view would show.
- [-] **`list_sessions` cannot search summaries.** Filters are `agent_id` / `tag` / `finalized` only;
  finding a session by what its summary says means paging. Parked 2026-09-09: substring `CONTAINS` on
  `summary` is one clause if wanted; semantic search would mean embedding summaries on finalize, which
  is a schema change and a separate task.

### Embedding migration follow-ups (deferred from the 2026-09-08 branch review)

- [-] **`CandleProvider::set_cls_pooling` is public API that exists only for one test** (2026-09-09).
  Integration tests cannot see `cfg(test)` items, so the hook is `pub` behind `#[doc(hidden)]`. A cargo
  feature gate (`test-util`, self dev-dependency) would hide it properly; add one if a second such hook
  appears.

### Retrieval benchmark follow-ups (2026-09-09)

- [-] **Rank inflates as the corpus grows; the data is logged, not tracked.** Five passes of the same
  12 questions: 143 facts `mean_rank` 1.42 / `top1` 9/12 / `mean_gap` +0.148; 743 -> 2.75, 7/12,
  +0.077; 807 -> 2.83; 830 -> 2.83, 7/12, `hit_min` 0.338 (830 is approximate — see the live-copy
  item below). Reframed 2026-09-09 under adversarial review, having been filed as "retrieval degrades
  as the corpus grows": one jump carries that whole claim, the last three passes are flat, and rank
  inflation under 5x more distractors is the null hypothesis rather than a finding. `hit_min` and
  `hit_max` are identical across every row, so absolute scores — which is what the client threshold
  actually filters on — have not moved at all. The only channel by which rank inflation reaches a
  user is `limit = 5` truncation, which is the `ALEXANDRIA_AUTO_RECALL_LIMIT` item below; act there,
  not here. Keep appending a pass at each significant corpus size, and reopen this as a defect only
  if `hit_min` starts moving.
- [ ] **The question set only targets facts from the original 143.** All 12 targets predate
  2026-09-08 16:26 UTC, so the 590 facts added since are never a correct answer, only distractors.
  That makes the third pass a clean measurement of "fixed questions against a growing haystack",
  which is what was wanted here, but it is not a measurement of retrieval quality on current
  material — nothing checks that a memory stored last week can be found at all. Add questions
  targeting recent facts before reading the bench as a general quality signal.
- [-] **Measured numbers are duplicated across docs and source, and nothing checks them** (2026-09-09).
  `docs/minilm-test-data.md`'s results section is a snapshot of a 743-fact corpus that grows every
  session, so its numbers go wrong-but-plausible rather than obviously wrong; nothing regenerates it
  and nothing compares it against a fresh `bench-retrieval` run, and there is no "as of" check beyond
  the date in the heading. The same headline numbers are repeated in the pointer at the end of
  `docs/plans/2026-09-08-embedding-model-swap-measurements.md`, and the retrieve-floor derivation
  lives in both `docs/configuration.md` and the `min_similarity` doc comment in `src/config.rs` —
  fixing only the file a TODO names leaves the stale claim live in the other. Parked: a CI job would
  need the live corpus (CI has none) and a discipline will not hold. Rule of thumb instead of a
  mechanism: when a measured number would appear in two places, put it in one and point at it from
  the other, and grep `src/config.rs` before closing any future "stale number in configuration.md"
  item.
- [-] **The `bench-retrieval` baseline pass is reconstructed by size, not recorded** (2026-09-09).
  `BASELINE_SIZE = 143` takes the 143 oldest active facts, which is not the same set as the 143 that
  were active on 2026-09-08. It reproduced the recorded table exactly this time. Two ways it can drift
  with nothing noticing, neither detectable from inside the tool: deleting a fact *inside* the window
  lets the window reach forward to replace it (deletions outside it are harmless — the "grows
  monotonically with every deletion" claim this item carried until 2026-09-09 was wrong), and
  `update_memory` keeps the record ID while rewriting the content, so any of the twelve frozen
  `QUESTIONS` targets can silently start measuring different text with every metric still looking
  comparable. The second is the likelier of the two, since the question set is frozen by ID and
  nothing about editing a memory warns you it is a benchmark target. A timestamp cutoff cannot be
  substituted: the measurements doc's "08:08 data copy" reads as UTC but is UTC-4 and is when the
  data dir was created — no active fact predates 2026-09-08 12:30 UTC, so a cutoff built from it
  selects nothing, which is how the size-based reconstruction came about (every other bare timestamp
  in these plan docs is local too). If the baseline row ever stops reproducing, suspect this before
  suspecting the metrics.
- [ ] **`bench-retrieval` silently truncates the corpus at 100,000 facts, oldest-first** (2026-09-09,
  found by adversarial review). `src/bench.rs` reads the corpus with
  `MemoryRepo::list(None, None, false, 100_000, 0)`, and `memory_repo.rs:146` orders
  `created_at DESC` — so past 100k facts `all` holds the *newest* 100k. The baseline window then
  stops being the original install entirely, and the frozen `QUESTIONS` targets, every one of them
  from the original 143, fall out of the corpus. The `scored == 0` bail only fires if all twelve
  vanish at once; losing four just shifts every metric with no signal at all. Same corruption as the
  baseline drift above, reached by growth instead of deletion. 120x away at 830 facts — and the
  corpus went 143 -> 830 in two days. The fix is a bail when `all.len()` reaches the cap, not a
  bigger cap.
- [-] **`bench-retrieval` sorts `created_at: None` facts to the front of the baseline window**
  (2026-09-09, found by the same review). `by_age.sort_by_key` in `src/bench.rs` keys on
  `Option<DateTime>`, and `None` orders before `Some`, so a fact with no timestamp would displace a
  genuine oldest fact from the 143. The adjacent `oldest`/`newest` log lines `filter_map` the `None`s
  away, so the function is inconsistent with itself about whether they can occur. Parked because they
  cannot: `v001_initial.surql:16` declares `created_at ON fact TYPE datetime DEFAULT time::now()`, so
  reaching this needs a direct write that bypasses the schema default. Recorded rather than fixed so
  the next reader of that sort does not have to re-derive why it is safe.
- [-] **The `bench-retrieval` corpus copy was taken from a live data dir** (2026-09-09). `README.md:55-57`
  says to stop the server for the `cp` and then run against the copy via `ALEXANDRIA_DATA_DIR`; the
  `scored == 0` verification skipped the stop and copied `~/.local/share/alexandria/data` while the
  server held the writer lock. SurrealKV replayed the WAL and reported 830 facts, and the metrics row
  matched the previous run, so nothing detectably tore — but an LSM tree copied mid-write is not
  guaranteed consistent, and a silently truncated copy would look like a corpus-size data point rather
  than an error. Treat the 830 figure as approximate. Stop the server for any copy whose numbers get
  recorded in `docs/minilm-test-data.md`.
- [ ] **`ALEXANDRIA_AUTO_RECALL_LIMIT` is an untuned lever and is the binding constraint for some
  questions** (2026-09-09, from the threshold sweep). At `T = 0.30` every one of the 12 targets clears
  the threshold on score, yet only 10 are delivered: two rank 8th and 7th, outside the hook's
  `limit = 5`, so no threshold can reach them. For those questions the threshold is irrelevant and the
  limit is the whole story. Nothing has ever measured what `limit` should be — 5 is the value both
  clients inherited unexamined. The sweep already computes `hits_delivered` against `RECALL_LIMIT`, so
  sweeping the limit instead of the threshold is a small change to `src/bench.rs`. Do that before
  touching the threshold again; raising the limit and lowering the threshold trade against each other
  and only one of the two has been measured. The rank-inflation item above folds into this one: rank
  only reaches a user through this limit.
- [-] **The pi extension default diverges from upstream behaviourally, and will merge without a
  conflict** (2026-09-09; premise corrected 2026-09-09). `recallMinSimilarity` in
  `contrib/pi/extensions/alexandria-auto-recall/src/config.ts` is `0.35` against upstream's `0.58`,
  which closes the "pending a change to the extension" note `docs/configuration.md` used to carry but
  makes a previously documentation-only divergence behavioural. This item claimed the file was a known
  merge surface, citing `docs/plans/2026-09-09-upstream-sync.md:31` — that line is
  `alexandria-auto-recall/README.md`, and `config.ts` appears nowhere in that plan: it did not
  conflict in the last sync. That is the risk, not the reassurance it reads as. A one-sided edit
  conflicts only if upstream touches the same lines, so absent a conflict there is no review point,
  and a blanket `:theirs` on any *other* conflict in that file reverts our value silently. Per the
  `:theirs` lesson below, grep `config.ts` for `0.35` after every sync whether or not the merge
  reported a conflict.

## Build / toolchain

- [-] **mmap was rejected for model loading; the ~90 MB double peak is accepted** (2026-09-08).
  `candle.rs` reads the safetensors file into a `Vec<u8>` (`from_buffered_safetensors`) instead of
  mmap, so the workspace can carry `unsafe_code = "forbid"`. The buffer and the built tensors coexist
  until `BertModel::load` returns, then the buffer drops; there is no earlier drop point in safe code.
  Neither alternative removes the peak — candle still copies each tensor out of an mmap (the source
  bytes just become reclaimable page cache), and streaming through the `safetensors` reader into a
  `HashMap` holds the same peak — so there is nothing to do at MiniLM's size. Kept as a `[-]` only
  because `candle.rs` carries no comment recording the choice; move it there and drop this. Escape
  hatch if a much larger model is adopted: `#[allow(unsafe_code)]` on that one call plus
  `from_mmaped_safetensors`.

- [-] **`.githooks/pre-commit` hard-depends on `just`** (2026-09-09, accepted when the hook was
  collapsed to `just fmt` / `just lint` and the hand-copied `cargo` invocations were deleted). A
  fallback to the literal commands would re-create the duplication the collapse removed, so there is
  none: without `just` on PATH the hook dies with `just: command not found`. Judged safe because the
  only documented install path is `just install-hooks` (justfile:45), which already requires it, and
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

- [-] **`tokenizers` is pinned by candle-core 0.11** (2026-09-08). Held at 0.22 to match (0.23 built
  a second copy), and candle-core depends on it with the `onig` feature itself, so our
  `default-features = false` cannot drop the C build. Both move together on the next candle bump that
  moves its own; `fancy-regex` is the pure-Rust alternative if the feature ever becomes ours to
  choose. Rechecked 2026-09-09: 0.11.0 (2026-06-26) is still the newest candle-core release.
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
  `contrib/claude/README.md:30` records the same non-port; keep the two in step or drop one.
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
