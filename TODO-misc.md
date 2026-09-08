# TODO (misc)

Open items noticed while getting Alexandria running under Claude Code (2026-09-08).

## Retrieval quality

- [ ] **Embedding model is the real ceiling.** `all-MiniLM-L6-v2` is a symmetric similarity
  model. A natural-language question against a stored statement scores only ~0.1–0.2
  cosine ("which database does the project use" vs "the project uses SurrealDB" = 0.19),
  while a keyword hit scores ~0.6. The floor was lowered to 0.10 to compensate, but an
  asymmetric retrieval model (e.g. an msmarco/bge/e5 family model) would separate real
  matches from noise far better. Blocked on: model is locked on first boot, so switching
  the default needs a migration/re-embed story.
  Measured 2026-09-08 on the live corpus (143 facts, 12 questions; see
  `docs/plans/2026-09-08-embedding-model-swap-measurements.md`): none of msmarco-MiniLM-L6-cos-v5,
  multi-qa-MiniLM-L6-cos-v1, or bge-small-en-v1.5 beat MiniLM (mean rank 1.42 vs 2.33 for the best
  challenger, mean gap +0.148 vs +0.091). Default unchanged. bge-small scores higher in absolute
  terms (hit_min 0.620 vs 0.338) but its noise floor rises just as much (nonhit_p50 0.564 vs 0.077),
  so separation is worse. The blocker is gone: `alexandria migrate-embeddings` re-embeds an existing
  database and CLS-pooled models load, so a future candidate is a config change plus one command.
  Note bge was measured without its query instruction prefix.

## Server

- [-] **Implicitly created sessions have no `agent_id` or `model`.** `SessionRepo::find_or_create`
  (2026-09-08, replaced the duplicated block in `do_store_memory` / `do_import_document`) creates with
  both `None`, and `finalize_session` only sets summary and tags, so a session first seen via
  `store_memory` can never acquire them. Same as before the refactor; nothing reads the fields yet.
  Add optional `agent_id`/`model` params to `store_memory` and thread them through if a session view
  ever wants them.
- [-] **`raw` record carries no session.** The 2026-09-08 `import_document` session linkage attaches
  the chunks only; the `raw` document record is reachable from them via `extracted_from` but has no
  session edge of its own. Parked 2026-09-08: `contains_session_memory` is declared `IN session OUT fact`,
  so linking `raw` needs a new edge table plus a schema migration, and nothing reads it. Add one if a
  session view ever needs the source document directly.

### Embedding migration follow-ups (deferred from the 2026-09-08 branch review)

- [ ] **`migrate-embeddings` has never run against a real SurrealKV database.** Only the fake-provider
  test on `kv-mem` and an empty scratch dir exercised it. Before first real use: stop the service,
  `cp -a` the data dir to `/tmp`, run it there with `ALEXANDRIA_DATA_DIR` pointed at the copy and
  `ALEXANDRIA_EMBEDDING_MODEL=sentence-transformers/multi-qa-MiniLM-L6-cos-v1`, then migrate the copy
  back to MiniLM. Throw the copy away.
- [x] **`migrate.rs` batch size is a hardcoded 32.** Done 2026-09-08: `embedding.batch_size` in
  `config.toml` (default 32, must be >= 1), passed to `reembed()`; `ALEXANDRIA_EMBEDDING_BATCH_SIZE` overrides it.
- [-] **`embedding.batch_size = 0` is rejected by `reembed()`, not at config load.** 2026-09-08: the
  server boots fine with 0 because nothing there reads the field; only `migrate-embeddings` errors.
  Same shape as `server.port` (parse errors caught at load, range errors at use). Move the check
  into `Config::load_from` if a second consumer of the field ever appears.
- [-] **`migrate-embeddings` no longer logs "Alexandria v0.2 starting..."** (2026-09-08, side effect of
  ed923ee): the subcommand returns from inside the argument match, before the startup log line. It
  still logs its own progress. Accepted; add a line at the top of `migrate_embeddings()` if it matters.
- [ ] **Spec defect: threshold-derivation rule has no valid solution when `nonhit_p99 > hit_min`.**
  The design spec's `retrieve.min_similarity` rule (and its midpoint fallback) lands above `hit_min`
  on this corpus (0.373 vs 0.338), so any derived floor cuts a true hit. Rewrite the rule before the
  next model bench (see `docs/plans/2026-09-08-embedding-model-swap-measurements.md`).
- [-] **`alexandria-pipeline` unit tests now need the real model** (2026-09-08, d4bc3eb). The CLS-vs-mean
  test sits in `candle.rs` under `#[cfg(test)]` because it flips the private pooling flag, so
  `cargo test -p alexandria-pipeline --lib` downloads MiniLM on a cold cache where before only the
  `tests/` integration tests did. Accepted; if it bothers anyone, expose a test-only constructor and
  move the test to `tests/embedding_test.rs` with the other slow ones.
- [x] **Server boot stamps the lock over an unlocked corpus.** Done 2026-09-08: `check_embedding_model`
  now counts facts before stamping and warns (facts present, no lock, model assumed) instead of
  stamping silently. It still stamps, because the `migrate-embeddings` guard tells the user to "start
  the server once" to recover a pre-lock database; refusing here would close that path.
- [-] **Boot could refuse an unlocked corpus instead of warning.** Parked 2026-09-08: needs an escape
  hatch (e.g. `migrate-embeddings --assume-model`) so a pre-lock database can still be stamped, and the
  population is almost certainly nonexistent. Add both together if one ever turns up.

## Build / toolchain

- [ ] **Windows `rustflags` (msvc/gnu/gnullvm targets) added 2026-09-08 but unverified.** `.cargo/config.toml`
  sets `target-cpu=x86-64-v2` for the three Windows targets alongside the Linux `mold` target; there's
  no Windows toolchain in this environment to cross-compile and confirm they take effect.
- [ ] **`[profile.release]` (lto = "thin", codegen-units = 1, strip) added 2026-09-08, never built.**
  Only `cargo build --workspace` (dev profile) has been run since the toolchain/profile changes,
  and the 2026-09-08 dependency major bumps (and the same-day `hf-hub` removal in favour of
  `hub.rs`) were likewise only dev-built and tested; do a `cargo build --release` smoke test before
  shipping a release artifact.

- [ ] **Startup memory doubled during model load (2026-09-08).** `candle.rs` now reads the safetensors
  file into a `Vec<u8>` (`from_buffered_safetensors`) instead of mmap, so the workspace can carry
  `unsafe_code = "forbid"`. For MiniLM (~90 MB) the buffer plus the built tensors coexist briefly at
  boot, then the buffer drops. Revisit only if a much larger model is adopted; the escape hatch is a
  `#[allow(unsafe_code)]` on that one call plus `from_mmaped_safetensors`.

## Dependencies

- [x] **`hf-hub` 1.0 is heavy for what we use.** Done 2026-09-08: replaced by
  `alexandria-pipeline/src/embedding/hub.rs` (~90 lines on `reqwest`, already in the tree via
  rmcp/surrealdb). Reads and writes the standard `models--owner--name/{refs/main,snapshots/<sha>/}`
  layout, so existing caches are reused with no network. Unique crates 658 -> 581.
- [-] **Cache-first model loading never refreshes a cached revision** (2026-09-08). Once `refs/main`
  is on disk it is served forever; delete `~/.cache/huggingface/hub/models--<owner>--<name>` to
  re-fetch. Accepted as-is.
- [-] **`hub.rs` shortcuts** (2026-09-08, all marked `ponytail:` in the file). Files go straight into
  `snapshots/<sha>/` with no `blobs/` symlink, no `.no_exist` marker, no lock files, no `HF_TOKEN`,
  no `HF_ENDPOINT`; only revision `main`. Consequences: a model that lacks `1_Pooling/config.json`
  pays one 404 per online boot and falls to the warn-and-assume-mean path offline (every
  sentence-transformers repo ships the file, so this never fires today); two servers first-booting
  on the same empty cache both download (rename-into-place keeps the result correct); gated or
  private models cannot be fetched. Add whichever one actually bites.
- [ ] **Two `tokenizers` versions compile** until candle bumps: candle-core 0.11 still pins 0.22, we're
  on 0.23. Behaviourally harmless; revert ours to 0.22 if the duplicate build cost bothers anyone.
- [ ] **Transitive "Unchanged" `cargo update` entries are upstream pins, not ours.** `generic-array`
  0.14.7, `i_float`/`i_overlay`/`i_shape`, `matchit` 0.8.4, `pdqselect` 0.1.0 stay put even after
  the direct bumps above; they move when the pulling crate (surrealdb stack) does.

## Claude Code integration

- [ ] **Error-resolution tracker not ported.** `contrib/claude/hooks/` now has auto-recall,
  session_id injection, correction/preference detectors, and Stop-hook LLM extraction
  (plan: `docs/plans/2026-09-08-todo-misc-plan.md`). The Pi error-resolution detector was
  deliberately skipped: it needs PostToolUse state across a turn and yields low-signal
  "Error with X / Resolution: <200 chars>" memories; the extraction pass captures root causes
  once resolved. Revisit only if extracted memories turn out to miss resolved errors.
- [ ] **Retry doubles cost on tactical turns.** Every turn where haiku correctly finds nothing now pays a
  second call (up to ~80 s wall, hidden by async). Watch the `extracted` volume; if it is mostly
  noise or the cost matters, drop the retry or gate it on transcript size.
- [ ] **Headless experiments write to the live server.** Any `claude -p` run on this machine fires the
  installed hooks, so a stub extractor's output (or haiku's) lands in the real database under a
  throwaway session id; 2026-09-08 testing left six `stub` memories that had to be deleted by hand.
  Prefix experiments with `ALEXANDRIA_AUTO_STORE=off` or point `ALEXANDRIA_URL` at a scratch server.
- [ ] **Hook development in a live interactive session pollutes the real database.** Companion to the
  headless item above: the installed Stop hook extracts from this session's transcript too, so stub
  payloads and probe strings from tests pasted into the conversation become `extracted` memories (a
  "Detach debug probe" memory from 2026-09-08 surfaced in auto-recall today). `test.sh` itself is
  clean (own session id, deletes on exit); the leak is the interactive session around it. Mitigation
  is the same: `ALEXANDRIA_AUTO_STORE=off` in the developing session's env, or delete by hand.
- [ ] **A queued follow-up prompt lands in the previous turn's chunk.** If the user types the next
  prompt while a turn is still generating, Claude Code dispatches it as soon as the turn ends, inside
  the 1 s flush wait, so the extract hook sees it with the previous turn. Harmless (it is extracted
  once, just one turn early); noted so it is not mistaken for a marker bug.
