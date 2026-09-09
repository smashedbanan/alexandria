# TODO (misc)

Open items noticed while getting Alexandria running under Claude Code (2026-09-08).

## Retrieval quality

- [-] **Model bench tooling is not in the tree** (2026-09-08). The candle bench example was deleted with
  the first pass and the second pass ran through a throwaway sentence-transformers script in
  `/tmp/alexandria-bench` (`uv run` with inline metadata pinning torch to the pytorch CPU index; corpus
  dumped to JSON by a one-off `MemoryRepo::list` example). Recipe is recorded in the measurements doc.
  Parked: the model question is closed, so nothing to keep.

## Server

- [x] Done 2026-09-08: **Claude Code hooks stamp `agent_id: "claude-code"`.** `alexandria-session.sh` now
  fills `agent_id` as well as `session_id` on `store_memory` / `import_document` calls that lack either
  (a value the model set is kept); the detector stores in `alexandria-recall.sh` and the extraction stores
  in `alexandria-extract.sh` send it directly. `SessionRepo::find_or_create` fills a still-empty field on
  an existing session, so every Claude Code session row gets stamped by its first store from any of the
  three. Still parked: `model` (not in the hook payload) and the Pi extension, which sends only
  `session_id`; add when a session view wants to tell those apart.
- [-] **`raw` record carries no session.** The 2026-09-08 `import_document` session linkage attaches
  the chunks only; the `raw` document record is reachable from them via `extracted_from` but has no
  session edge of its own. Parked 2026-09-08: `contains_session_memory` is declared `IN session OUT fact`,
  so linking `raw` needs a new edge table plus a schema migration, and nothing reads it. Add one if a
  session view ever needs the source document directly.

### Embedding migration follow-ups (deferred from the 2026-09-08 branch review)

- [x] Done 2026-09-08: **`embedding.batch_size = 0` is rejected at config load.** The `ensure!` moved
  from `reembed()` into `Config::load_from`, after the env overrides, so a bad TOML value or
  `ALEXANDRIA_EMBEDDING_BATCH_SIZE=0` refuses to boot instead of failing only under `migrate-embeddings`.
  `reembed()` no longer guards the value itself (`chunks(0)` would panic), so any future caller other
  than `main` has to pass a loaded config value; the `reembed_rejects_zero_batch_size` test went with it.
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

- [-] **The `hub.rs` download itself is still unexercised by the service** (2026-09-08). The service reinstall
  (f996e35) only proved the cache-first branch, since `~/.cache/huggingface/hub/models--sentence-transformers--all-MiniLM-L6-v2`
  was already populated. The download branch has only run under `cargo test`. To exercise it for real:
  move that directory aside, restart the service, confirm the fetch in the journal, then delete the
  moved copy. Parked; do it the next time the cache is wiped anyway.

- [-] **Startup memory doubled during model load (2026-09-08).** `candle.rs` reads the safetensors
  file into a `Vec<u8>` (`from_buffered_safetensors`) instead of mmap, so the workspace can carry
  `unsafe_code = "forbid"`. For MiniLM (~90 MB) the buffer and the built tensors coexist until
  `BertModel::load` returns, then the buffer drops; there is no earlier drop point in safe code.
  Parked 2026-09-08: mmap would not remove the copy either (candle still copies each tensor out of
  the map; the source bytes just become reclaimable page cache), and streaming tensors through the
  `safetensors` reader into a `HashMap` holds the same peak. Revisit only if a much larger model is
  adopted; the escape hatch is a `#[allow(unsafe_code)]` on that one call plus
  `from_mmaped_safetensors`.

- [x] Done 2026-09-09: **The installed service predates the `batch_size` boot check.** Rebuilt with
  `cargo install --path crates/alexandria` from 255dde7 and restarted the user unit; the installed binary
  now refuses `ALEXANDRIA_EMBEDDING_BATCH_SIZE=0` with "embedding.batch_size must be at least 1" before
  opening the database.

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
- [-] **Every dependency is `default-features = false` with features listed explicitly** (2026-09-08).
  surrealdb carries only `kv-mem` and `kv-surrealkv`; `protocol-ws` and `rustls` must be re-added
  (comment in the root `Cargo.toml`) if SurrealDB is ever not on localhost. Runtime-only defaults
  dropped on purpose: tokio `full` (six named features instead), tracing-subscriber `smallvec`, axum
  `tracing`/`tower-log`, tokenizers `progressbar`/`esaxx_fast`, base64 `simd-unsafe`, chrono
  `oldtime`/`wasmbind`, toml `display`. Kept on purpose: tracing-subscriber `ansi` and `tracing-log`,
  since dropping either changes log output without failing any test.
- [-] **`tokenizers` still builds `onig`** (2026-09-08). candle-core 0.11 depends on tokenizers with
  the `onig` feature itself, so our `default-features = false` cannot drop the C build. Goes away
  only if a candle bump drops it; `fancy-regex` is the pure-Rust alternative if it ever becomes ours
  to choose.
- [-] **`tokenizers` is held at 0.22 to match candle-core 0.11** (2026-09-08; was 0.23, which built a
  second copy). Bump the workspace pin together with the next candle bump that moves its own.
- [ ] **Transitive "Unchanged" `cargo update` entries are upstream pins, not ours.** `generic-array`
  0.14.7, `i_float`/`i_overlay`/`i_shape`, `matchit` 0.8.4, `pdqselect` 0.1.0 stay put even after
  the direct bumps above; they move when the pulling crate (surrealdb stack) does.

## Claude Code integration

- [-] **Extraction sees failed tool results, but not which tool or whether the retry worked** (2026-09-08,
  replaces the "port the Pi error-resolution tracker" item). `alexandria-extract.sh` now serializes
  `is_error` tool results as `[Tool error]: <first 300 chars>` next to the user/assistant text, minus
  `<tool_use_error>` harness refusals, and leaves pairing and root-cause judgement to haiku. Accepted
  noise: permission denials, worktree-isolation refusals, and user rejections still go in (each is a
  one-liner; the prompt already excludes common knowledge). Not done: tool-name attribution, which
  needs the `tool_use_id` joined back to the previous assistant line, and the Pi `error-resolution`
  tag. Add the filter if junk memories of that shape ever appear; add attribution if haiku's output
  turns out to need it. Error text counts toward `ALEXANDRIA_EXTRACT_MIN_CHARS`, so error-heavy
  sessions extract a turn earlier.
- [x] Done 2026-09-08: **All three hooks read stdin before any guard.** `alexandria-recall.sh` and
  `alexandria-session.sh` now match `alexandria-extract.sh`; the recall hook skips the read in debug CLI
  mode (`$# -gt 0`) so a one-shot tool call from a terminal does not block. The `hook()` helper in
  `test.sh` was already a bare `jq | hook` pipeline under `set -eo pipefail`, so the SIGPIPE hazard the
  parked entry described was live there too.
- [-] **Stop-hook extraction makes one haiku call per turn** (2026-09-08, retry dropped). The retry on an
  empty first result rested on one observation (empty, then three memories on the same prompt) and
  doubled the cost of every tactical turn; the extract log showed only the second call failing, on
  the shrunken timeout. If `extracted` volume drops noticeably, restore the loop gated on transcript
  size rather than unconditionally.
- [-] **The `sdk-*` gate does not cover other non-interactive entrypoints** (2026-09-08). The 2.1.263 binary
  also knows `claude-code-github-action`, `local-agent`, `remote`, `remote_cowork`, `remote_baku`, and
  `bench`, none of which match `sdk-*`, so auto-store stays on there. Nothing here runs in those surfaces
  yet. Widen the `case` in `alexandria-recall.sh` / `alexandria-extract.sh` if one is ever used with these
  hooks installed.
- [-] **Hook development in a live interactive session pollutes the real database.** Companion to the
  `CLAUDE_CODE_ENTRYPOINT` item: the installed Stop hook extracts from this session's transcript too, so stub
  payloads and probe strings from tests pasted into the conversation become `extracted` memories (a
  "Detach debug probe" memory from 2026-09-08 surfaced in auto-recall today). `test.sh` itself is
  clean (own session id, deletes on exit); the leak is the interactive session around it. Parked
  2026-09-08: nothing in the hook can tell a pasted stub payload from a real conversation, so the
  headless fix does not apply. Accepted mitigation: start the developing session with
  `ALEXANDRIA_AUTO_STORE=off`, or delete by hand afterwards.
- [-] **`extract.log` is append-only and never rotated** (2026-09-08). The detached extract copy appends
  its stderr to `$XDG_RUNTIME_DIR/alexandria/extract.log` for the life of the login session, and a hook
  file edited while a detached copy is mid-run leaves stale noise there (today: three "LLM call 2
  failed" lines and a line-118 syntax error from a half-written edit, none reproducible by the
  committed script). Truncated by hand 2026-09-08. Rotate only if it ever grows past a few KB.
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
- [x] Done 2026-09-08: **`shellcheck contrib/claude/hooks/*.sh` exits 0.** A `# shellcheck disable=SC2016`
  on the fence-stripping `sed` line in `alexandria-extract.sh`; the backticks there are a regex, not an
  unexpanded command substitution.
- [ ] **A queued follow-up prompt lands in the previous turn's chunk.** If the user types the next
  prompt while a turn is still generating, Claude Code dispatches it as soon as the turn ends, inside
  the 1 s flush wait, so the extract hook sees it with the previous turn. Harmless (it is extracted
  once, just one turn early); noted so it is not mistaken for a marker bug.
