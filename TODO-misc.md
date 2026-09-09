# TODO (misc)

Open items noticed while getting Alexandria running under Claude Code (2026-09-08).

## Retrieval quality

- [-] **Model bench tooling is not in the tree** (2026-09-08). The candle bench example was deleted with
  the first pass and the second pass ran through a throwaway sentence-transformers script in
  `/tmp/alexandria-bench` (`uv run` with inline metadata pinning torch to the pytorch CPU index; corpus
  dumped to JSON by a one-off `MemoryRepo::list` example). Recipe is recorded in the measurements doc.
  Parked: the model question is closed, so nothing to keep.

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

- [-] **`embedding.batch_size = 0` is rejected by `reembed()`, not at config load.** 2026-09-08: the
  server boots fine with 0 because nothing there reads the field; only `migrate-embeddings` errors.
  Same shape as `server.port` (parse errors caught at load, range errors at use). Move the check
  into `Config::load_from` if a second consumer of the field ever appears.
- [-] **`migrate-embeddings` no longer logs "Alexandria v0.2 starting..."** (2026-09-08, side effect of
  ed923ee): the subcommand returns from inside the argument match, before the startup log line. It
  still logs its own progress. Accepted; add a line at the top of `migrate_embeddings()` if it matters.
- [-] **The rewritten floor rule has only been applied on paper** (2026-09-08). `nonhit_p50` and the
  `< hit_min` check were read off the existing measurements tables; no script computes them, since the
  bench tooling is not in the tree (see the parked entry under Retrieval quality). The next bench
  should derive the value from its own output and confirm the table above.
- [-] **`config.rs` and `docs/configuration.md` describe `min_similarity` as 0.10 with no pointer to the
  derivation rule** (2026-09-08). The rule gives 0.08 for MiniLM; 0.10 was kept as-is because the
  incumbent won and the difference is immaterial. If a model swap ever lands, the doc comment and the
  configuration table should cite the rule in the design plan rather than restating measured ranges.
- [-] **`alexandria-pipeline` unit tests now need the real model** (2026-09-08, d4bc3eb). The CLS-vs-mean
  test sits in `candle.rs` under `#[cfg(test)]` because it flips the private pooling flag, so
  `cargo test -p alexandria-pipeline --lib` downloads MiniLM on a cold cache where before only the
  `tests/` integration tests did. Accepted; if it bothers anyone, expose a test-only constructor and
  move the test to `tests/embedding_test.rs` with the other slow ones.
- [-] **Boot could refuse an unlocked corpus instead of warning.** Parked 2026-09-08: needs an escape
  hatch (e.g. `migrate-embeddings --assume-model`) so a pre-lock database can still be stamped, and the
  population is almost certainly nonexistent. Add both together if one ever turns up.

## Build / toolchain

- [ ] **Windows `rustflags` (msvc/gnu/gnullvm targets) added 2026-09-08 but unverified.** `.cargo/config.toml`
  sets `target-cpu=x86-64-v2` for the three Windows targets alongside the Linux `mold` target; there's
  no Windows toolchain in this environment to cross-compile and confirm they take effect.
- [-] **The `hub.rs` download itself is still unexercised by the service** (2026-09-08). The service reinstall
  (f996e35) only proved the cache-first branch, since `~/.cache/huggingface/hub/models--sentence-transformers--all-MiniLM-L6-v2`
  was already populated. The download branch has only run under `cargo test`. To exercise it for real:
  move that directory aside, restart the service, confirm the fetch in the journal, then delete the
  moved copy. Parked; do it the next time the cache is wiped anyway.

- [ ] **Startup memory doubled during model load (2026-09-08).** `candle.rs` now reads the safetensors
  file into a `Vec<u8>` (`from_buffered_safetensors`) instead of mmap, so the workspace can carry
  `unsafe_code = "forbid"`. For MiniLM (~90 MB) the buffer plus the built tensors coexist briefly at
  boot, then the buffer drops. Revisit only if a much larger model is adopted; the escape hatch is a
  `#[allow(unsafe_code)]` on that one call plus `from_mmaped_safetensors`.

## Dependencies

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
- [ ] **`CLAUDE_CODE_ENTRYPOINT` is undocumented** (2026-09-08). The hooks default auto-store off
  when this internal env var matches `sdk-*`; observed on Claude Code 2.1.263 (`cli` interactive,
  `sdk-cli` for `claude -p`).
  The `sdk-*` glob is meant to also cover Agent SDK harnesses, but those values are assumed, not
  observed: third-party harnesses and the Agent SDK run on API billing only, and the subscription
  plan cannot drive them, so they cannot be checked from this machine. If a release renames the
  variable the hooks silently fall back to the old always-on behaviour; after Claude Code upgrades,
  re-run a throwaway `claude -p` and confirm no session or marker files appear.
- [-] **Hook development in a live interactive session pollutes the real database.** Companion to the
  `CLAUDE_CODE_ENTRYPOINT` item: the installed Stop hook extracts from this session's transcript too, so stub
  payloads and probe strings from tests pasted into the conversation become `extracted` memories (a
  "Detach debug probe" memory from 2026-09-08 surfaced in auto-recall today). `test.sh` itself is
  clean (own session id, deletes on exit); the leak is the interactive session around it. Parked
  2026-09-08: nothing in the hook can tell a pasted stub payload from a real conversation, so the
  headless fix does not apply. Accepted mitigation: start the developing session with
  `ALEXANDRIA_AUTO_STORE=off`, or delete by hand afterwards.
- [ ] **A queued follow-up prompt lands in the previous turn's chunk.** If the user types the next
  prompt while a turn is still generating, Claude Code dispatches it as soon as the turn ends, inside
  the 1 s flush wait, so the extract hook sees it with the previous turn. Harmless (it is extracted
  once, just one turn early); noted so it is not mistaken for a marker bug.
