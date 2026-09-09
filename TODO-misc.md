# TODO (misc)

Open items noticed while getting Alexandria running under Claude Code (2026-09-08).

## Server

- [-] **`raw` record carries no session.** The 2026-09-08 `import_document` session linkage attaches
  the chunks only; the `raw` document record is reachable from them via `extracted_from` but has no
  session edge of its own. Parked 2026-09-08: `contains_session_memory` is declared `IN session OUT fact`,
  so linking `raw` needs a new edge table plus a schema migration, and nothing reads it. Add one if a
  session view ever needs the source document directly.

### Embedding migration follow-ups (deferred from the 2026-09-08 branch review)

- [-] **The rewritten floor rule has only been applied on paper** (2026-09-08). `nonhit_p50` and the
  `< hit_min` check were read off the existing measurements tables; no script computes them. The bench
  tooling is not in the tree: the candle bench example was deleted with the first pass and the second
  pass ran through a throwaway sentence-transformers script in `/tmp/alexandria-bench`, recipe recorded
  in the measurements doc. Accepted as-is 2026-09-09: the model question is closed, so no bench is
  planned. If one is ever rerun, derive the floor from its own output and confirm the table.
- [-] **The `AlexandriaServer` builder default for `retrieve_min_similarity` is kept equal to
  `RetrieveConfig` by hand** (2026-09-09, both 0.10). No test asserts they match: the config type lives
  in the binary crate and the builder in `alexandria-mcp`, and `main.rs` always overrides the builder
  from config, so the builder value only reaches tests. Add an assertion in `src/config.rs` tests if it
  ever drifts again.
- [-] **`CandleProvider::set_cls_pooling` is public API that exists only for one test** (2026-09-09).
  Integration tests cannot see `cfg(test)` items, so the hook is `pub` behind `#[doc(hidden)]`. A cargo
  feature gate (`test-util`, self dev-dependency) would hide it properly; add one if a second such hook
  appears.
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
  `[Tool error]: <tool> <first 120 chars of its input> -- <first 300 chars>`, joined through
  `tool_use_id` (a result whose call is not in the chunk keeps the bare form), and leaves pairing
  and root-cause judgement to haiku. Accepted noise: permission denials, worktree-isolation refusals,
  and user rejections still go in. Add the tag filter only if junk memories of that shape appear.
- [ ] **Tool input in `[Tool error]` lines is cut at 120 characters of raw JSON** (2026-09-09), so a
  long Bash command truncates mid-string and the `description` field is usually lost. Readable enough
  for haiku today; the fix is `input.command // input.file_path` per tool. Do it or accept it for good
  and drop this entry.
- [-] **Stop-hook extraction makes one haiku call per turn** (2026-09-08, retry dropped). The retry on an
  empty first result rested on one observation (empty, then three memories on the same prompt) and
  doubled the cost of every tactical turn; the extract log showed only the second call failing, on
  the shrunken timeout. If `extracted` volume drops noticeably, restore the loop gated on transcript
  size rather than unconditionally.
- [ ] **The `sdk-*` gate does not cover other non-interactive entrypoints** (2026-09-08). The 2.1.263 binary
  also knows `claude-code-github-action`, `local-agent`, `remote`, `remote_cowork`, `remote_baku`, and
  `bench`, none of which match `sdk-*`, so auto-store stays on there. Nothing here runs in those surfaces
  yet, but widening the `case` is one line each in `alexandria-recall.sh:108` and
  `alexandria-extract.sh:27`.
- [-] **Hook development in a live interactive session pollutes the real database.** Companion to the
  `CLAUDE_CODE_ENTRYPOINT` item: the installed Stop hook extracts from this session's transcript too, so stub
  payloads and probe strings from tests pasted into the conversation become `extracted` memories (a
  "Detach debug probe" memory from 2026-09-08 surfaced in auto-recall today). `test.sh` itself is
  clean (own session id, deletes on exit); the leak is the interactive session around it. Parked
  2026-09-08: nothing in the hook can tell a pasted stub payload from a real conversation, so the
  headless fix does not apply. Accepted mitigation: start the developing session with
  `ALEXANDRIA_AUTO_STORE=off`, or delete by hand afterwards.
- [ ] **Marker pruning runs only from the Stop hook** (2026-09-09). `.stored` markers from the recall hook
  are only pruned when a Stop hook fires on the same machine. Fine as long as one Claude Code install is
  in play; the fix is copying the `find` from `alexandria-extract.sh:45` into `alexandria-recall.sh`.
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
- [ ] **A queued follow-up prompt lands in the previous turn's chunk.** If the user types the next
  prompt while a turn is still generating, Claude Code dispatches it as soon as the turn ends, inside
  the 1 s flush wait, so the extract hook sees it with the previous turn. Harmless (it is extracted
  once, just one turn early); noted so it is not mistaken for a marker bug.
