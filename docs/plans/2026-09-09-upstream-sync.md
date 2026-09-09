# Upstream Sync (cebarks/alexandria main → claude2boogaloo) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land the 13 docs+CI commits from `cebarks/alexandria` main (a4b7f38..9c46ce0) on `claude2boogaloo` as one jj merge change, with the five doc conflicts resolved and upstream's `AGENTS.md` corrected for this fork.

**Architecture:** One merge change with parents `claude2boogaloo` (our 85 commits) and `main` (local bookmark already at upstream tip 9c46ce0). No history is rewritten. Upstream's `CLAUDE.md → AGENTS.md` rename is accepted; `CLAUDE.md` becomes a one-line `@AGENTS.md` import so Claude Code still loads it. Conflicted hunks are resolved with `jj resolve --tool :theirs` (side 2 = `main` = upstream) and then our fork-specific lines are re-added by edit.

**Tech Stack:** jj 0.45 (colocated with git), `just ci` (fmt + clippy + test + cargo-deny).

**Spec:** Design agreed in chat 2026-09-09 (this plan is the only written artifact). Summary of the agreed rules is the "Resolution rules" section below.

## Global Constraints

- Version control: use `jj`, not `git`, for every state change (per repo convention). `git` is fine read-only (`git merge-tree`, `git show`).
- Parent order matters: `jj new claude2boogaloo main` makes side #1 = ours, side #2 = upstream. `:ours` / `:theirs` in `jj resolve` refer to those sides.
- Upstream has zero Rust changes. If `jj st` ever lists a conflict outside the five files below, stop and report.
- Nothing under `.github/`, `docs/session-memory.md`, or `docs/roadmap.md` is hand-edited. They auto-merge.
- The 10 historical `docs/plans/2025-*` and `docs/plans/2026-08-*` files we previously deleted stay deleted.
- `origin/main` is not touched. Only `claude2boogaloo` moves.

## Resolution rules (agreed design)

| File | Rule |
|---|---|
| `CLAUDE.md` (modify/delete) | Accept deletion of the old content; write a new one-line `CLAUDE.md` containing `@AGENTS.md`. |
| `AGENTS.md` (new from upstream, no conflict) | Re-add our Build Gate section and the `record_id_to_string` location detail. Correct stale claims: migrations head is `v006`, `touch()` no longer bumps `memory_count`, the "known inconsistency" about session retrieval leaking deleted memories is already fixed here, test count is re-measured. Note in Docs Map that `CLAUDE.md` is an import shim. |
| `README.md` | Take upstream's pi-extension paragraph, append our Claude Code hook pointer sentence. |
| `docs/configuration.md` | Union of env vars: upstream's three `ALEXANDRIA_SERVER_*` plus our `ALEXANDRIA_EMBEDDING_BATCH_SIZE`. |
| `contrib/pi/README.md` | Take upstream's `client.toml` paragraph, add one sentence with our 0.35 threshold recommendation. |
| `contrib/pi/extensions/alexandria-auto-recall/README.md` | Take upstream's table rows, keep our 0.35 note on the `MIN_SIMILARITY` row. |

---

### Task 1: Create the merge change and confirm the conflict set

**Files:**
- No edits. Creates a new jj change.

**Interfaces:**
- Produces: working copy `@` = merge of `claude2boogaloo` and `main`, with exactly five conflicted paths.

- [ ] **Step 1: Confirm starting state**

Run:
```bash
jj log -r 'claude2boogaloo | main' --no-graph -T 'bookmarks ++ " " ++ commit_id.short() ++ " " ++ description.first_line() ++ "\n"'
```
Expected:
```
claude2boogaloo b876a2be fix(contrib/claude): end the extract chunk at the last assistant line
main 9c46ce0c docs: describe the container workflow and its timing profile
```
If `main` is not at `9c46ce0c`, run `jj git fetch --remote upstream` and `jj bookmark set main -r main@upstream`, then re-check.

- [ ] **Step 2: Create the merge change**

Run:
```bash
jj new claude2boogaloo main -m "merge: sync upstream cebarks/alexandria main (docs + CI, a4b7f38..9c46ce0)"
```

- [ ] **Step 3: Verify the conflict set is exactly the expected five files**

Run:
```bash
jj resolve --list
```
Expected (order may differ):
```
CLAUDE.md    2-sided conflict including 1 deletion
README.md    2-sided conflict
contrib/pi/README.md    2-sided conflict
contrib/pi/extensions/alexandria-auto-recall/README.md    2-sided conflict
docs/configuration.md    2-sided conflict
```
Any other path listed → stop and report; do not continue.

- [ ] **Step 4: Verify CI files and new docs auto-merged from upstream**

Run:
```bash
jj diff --from main --stat -- .github docs/session-memory.md docs/roadmap.md AGENTS.md
```
Expected: no output (identical to upstream).

---

### Task 2: Resolve `CLAUDE.md` and correct `AGENTS.md`

**Files:**
- Modify: `CLAUDE.md` (replace entire content)
- Modify: `AGENTS.md` (lines 14, 46–48, 54, 89; insert Build Gate section before `## Testing`)

**Interfaces:**
- Produces: `CLAUDE.md` == `@AGENTS.md\n`; `AGENTS.md` with the corrections listed in the rules table.

- [ ] **Step 1: Resolve the CLAUDE.md conflict by writing the import shim**

Run:
```bash
printf '@AGENTS.md\n' > CLAUDE.md
jj resolve --list
```
Expected: `CLAUDE.md` no longer listed (writing a non-conflicted file resolves it).

- [ ] **Step 2: Restore the `record_id_to_string` location detail (line 14)**

Replace in `AGENTS.md`:
```
- `RecordId` formatting: use `record_id_to_string()` helper, not `.to_string()`
```
with:
```
- `RecordId` formatting: use `record_id_to_string()` helper (in `alexandria-storage/src/lib.rs`, re-exported by `alexandria-mcp/src/server.rs`), not `.to_string()`
```

- [ ] **Step 3: Correct the session `touch()` claim (line 46)**

Replace:
```
- Sessions are created implicitly by `store_memory(session_id)` — there is no create tool. `SessionRepo::touch()` bumps `memory_count` **and** `ended_at`, so `ended_at` means last-activity; only a non-null `summary` distinguishes a finalized session. See `docs/session-memory.md`.
```
with:
```
- Sessions are created implicitly by `store_memory(session_id)` — there is no create tool. `SessionRepo::touch()` bumps `ended_at`, so `ended_at` means last-activity; only a non-null `summary` distinguishes a finalized session. `memory_count` is computed live, not stored (`v006` dropped the column). See `docs/session-memory.md`.
```

- [ ] **Step 4: Delete the "Known inconsistency" bullet (line 47)**

Delete this entire line from `AGENTS.md`. Our `SessionRepo::get_memories()` already filters `WHERE deleted = false` (see `crates/alexandria-storage/src/repos/session_repo.rs:141`), so the claim is false for this fork.
```
- Known inconsistency: session-scoped retrieval walks edges through `SessionRepo::get_memories()` → `MemoryRepo::get_fact()`, which does **not** filter `deleted = false` the way the unscoped path does. Soft-deleted memories therefore still surface in `get_session` and `retrieve_memories(session_id: ...)`. Not yet fixed — don't document it as intended behavior.
```

- [ ] **Step 5: Correct the migrations head (line 48)**

Replace `Current head is \`v005_session.surql\`.` with `Current head is \`v006_drop_session_memory_count.surql\`.`

- [ ] **Step 6: Re-measure the test count and correct line 54**

Run:
```bash
cargo test --workspace 2>&1 | grep -E '^test result' | awk '{s+=$4} END{print s}'
```
Note the number N. Replace `Current suite: 116 tests, all green.` with `Current suite: N tests, all green.` using the measured N. (First run downloads the ~80MB MiniLM model; allow up to 10 minutes.)

- [ ] **Step 7: Insert the Build Gate section before `## Testing`**

Insert the following block immediately before the line `## Testing` in `AGENTS.md`:
```markdown
## Build Gate

Before any `cargo check`, `cargo run`, or `cargo build`, run these in order and fix what they report:

1. `cargo fmt --check`
2. `cargo clippy -- -D warnings`

Both must pass clean first. Do not skip the gate to "just see if it compiles".

```

- [ ] **Step 8: Update the Docs Map self-reference (line 89 area)**

Replace:
```
- `AGENTS.md` — this file. It was named `CLAUDE.md` until the docs sweep that added session memory
  and the pi extension docs, so older `docs/plans/*` references to `CLAUDE.md` point here.
```
with:
```
- `AGENTS.md` — this file. It was named `CLAUDE.md` until the docs sweep that added session memory
  and the pi extension docs, so older `docs/plans/*` references to `CLAUDE.md` point here.
  `CLAUDE.md` still exists as a one-line `@AGENTS.md` import so Claude Code auto-loads this file.
```

- [ ] **Step 9: Verify**

Run:
```bash
cat CLAUDE.md
grep -c 'v005_session' AGENTS.md; grep -c 'Known inconsistency' AGENTS.md; grep -c '116 tests' AGENTS.md
grep -c '^## Build Gate' AGENTS.md; grep -c 'v006_drop_session_memory_count' AGENTS.md; grep -c '@AGENTS.md' AGENTS.md
```
Expected: `@AGENTS.md`, then `0 0 0`, then `1 1 1`.

---

### Task 3: Resolve `README.md`

**Files:**
- Modify: `README.md` (conflict at the "Optional pi extension" list item, ~line 79–94)

- [ ] **Step 1: Take upstream's side of the conflicted hunk**

Run:
```bash
jj resolve --tool :theirs README.md
```

- [ ] **Step 2: Append our Claude Code pointer to the paragraph**

Find the list item that ends with:
```
    recall and much denser capture.
```
and change it to:
```
    recall and much denser capture. The Claude Code equivalent of the recall half is a
    `UserPromptSubmit` hook at
    [`contrib/claude/hooks/alexandria-recall.sh`](contrib/claude/hooks/alexandria-recall.sh).
```

- [ ] **Step 3: Verify both sides survived**

Run:
```bash
grep -c 'alexandria-recall.sh' README.md
grep -c 'heuristic detectors' README.md
grep -c 'journald.conf.d/alexandria.conf' README.md
grep -c '^### Session memory' README.md
grep -c '^### In Docker' README.md
```
Expected: every count ≥ 1.

---

### Task 4: Resolve `docs/configuration.md`

**Files:**
- Modify: `docs/configuration.md` (line ~11, the "Individual env vars" list item)

- [ ] **Step 1: Take upstream's side**

Run:
```bash
jj resolve --tool :theirs docs/configuration.md
```

- [ ] **Step 2: Add our batch-size env var to the list**

Replace:
```
3. **Individual env vars** — `ALEXANDRIA_SERVER_TRANSPORT`, `ALEXANDRIA_SERVER_HOST`, `ALEXANDRIA_SERVER_PORT`, `ALEXANDRIA_DATA_DIR`, `ALEXANDRIA_EMBEDDING_MODEL`, `ALEXANDRIA_EMBEDDING_DEVICE`
```
with:
```
3. **Individual env vars** — `ALEXANDRIA_SERVER_TRANSPORT`, `ALEXANDRIA_SERVER_HOST`, `ALEXANDRIA_SERVER_PORT`, `ALEXANDRIA_DATA_DIR`, `ALEXANDRIA_EMBEDDING_MODEL`, `ALEXANDRIA_EMBEDDING_DEVICE`, `ALEXANDRIA_EMBEDDING_BATCH_SIZE`
```

- [ ] **Step 3: Verify**

Run:
```bash
grep -c 'ALEXANDRIA_EMBEDDING_BATCH_SIZE' docs/configuration.md
grep -c 'ALEXANDRIA_SERVER_TRANSPORT' docs/configuration.md
grep -c '^\[recall\]' docs/configuration.md
```
Expected: every count ≥ 1 (the `[recall]` section is our earlier addition and must still be present).

---

### Task 5: Resolve the two pi READMEs

**Files:**
- Modify: `contrib/pi/README.md` (Configuration section, ~line 80)
- Modify: `contrib/pi/extensions/alexandria-auto-recall/README.md` (env var table, ~line 52)

- [ ] **Step 1: Take upstream's side of both**

Run:
```bash
jj resolve --tool :theirs contrib/pi/README.md
jj resolve --tool :theirs contrib/pi/extensions/alexandria-auto-recall/README.md
```

- [ ] **Step 2: Add the threshold recommendation to `contrib/pi/README.md`**

After the paragraph ending `the defaults work against a locally running server.` add a blank line and this paragraph:
```
The default `min_similarity` (0.58) was measured too high for `all-MiniLM-L6-v2`; set `0.35`
(see `[recall]` in [docs/configuration.md](../../docs/configuration.md)).
```

- [ ] **Step 3: Keep our note on the `MIN_SIMILARITY` row of the auto-recall README**

Replace:
```
| `ALEXANDRIA_AUTO_RECALL_MIN_SIMILARITY` | `0.58` | Minimum cosine similarity, inclusive (model-dependent; sits above the server-side `[retrieve] min_similarity` floor) |
```
with:
```
| `ALEXANDRIA_AUTO_RECALL_MIN_SIMILARITY` | `0.58` | Minimum cosine similarity, inclusive (model-dependent; sits above the server-side `[retrieve] min_similarity` floor). Measured too high for `all-MiniLM-L6-v2`: recommended `0.35`, see `[recall]` in `docs/configuration.md` |
```

- [ ] **Step 4: Verify**

Run:
```bash
grep -c '0.35' contrib/pi/README.md contrib/pi/extensions/alexandria-auto-recall/README.md
grep -c 'client.toml' contrib/pi/README.md
jj resolve --list
```
Expected: `1` for each README on the first command, ≥1 on the second, and `jj resolve --list` prints nothing (or "No conflicts found").

---

### Task 6: Verify the merged tree

**Files:**
- No edits.

- [ ] **Step 1: No conflict markers anywhere**

Run:
```bash
jj resolve --list
grep -rln '^<<<<<<<\|^%%%%%%%\|^>>>>>>>' --include='*.md' --include='*.yml' --include='*.toml' . | grep -v '^./target' || echo CLEAN
```
Expected: no conflicts; `CLEAN`.

- [ ] **Step 2: CI files are byte-identical to upstream**

Run:
```bash
jj diff --from main --stat -- .github docs/session-memory.md
```
Expected: no output.

- [ ] **Step 3: Full CI gate**

Run:
```bash
just ci
```
Expected: fmt, lint, test, deny all pass. If `cargo deny` fails on an advisory that upstream's `deny.toml` doesn't cover, report it; do not edit `deny.toml` in this merge.

- [ ] **Step 4: Claude Code loads AGENTS.md through the shim**

Run:
```bash
claude -p 'Quote the first line of the Build Gate section from your project instructions. Reply with that line only.'
```
Expected: the reply contains `Before any \`cargo check\``. If it doesn't, the `@AGENTS.md` import isn't being picked up; check `CLAUDE.md` is exactly `@AGENTS.md` followed by a newline.

- [ ] **Step 5: Review the merge diff against our side**

Run:
```bash
jj diff --from claude2boogaloo --stat
```
Expected: only these paths change relative to our branch:
```
.github/dependabot.yml
.github/workflows/ci.yml
.github/workflows/container.yml
AGENTS.md
CLAUDE.md
README.md
contrib/pi/README.md
contrib/pi/extensions/alexandria-auto-recall/README.md
contrib/pi/skills/alexandria-memory/SKILL.md
docs/configuration.md
docs/roadmap.md
docs/session-memory.md
```
Any Rust file listed → stop and report.

---

### Task 7: Move the bookmark and push

**Files:**
- No edits.

- [ ] **Step 1: Confirm the merge change has a description**

Run:
```bash
jj log -r @ --no-graph -T 'description.first_line() ++ "\n"'
```
Expected: `merge: sync upstream cebarks/alexandria main (docs + CI, a4b7f38..9c46ce0)`

- [ ] **Step 2: Move `claude2boogaloo` onto the merge**

Run:
```bash
jj bookmark set claude2boogaloo -r @
jj log -r 'claude2boogaloo' --no-graph -T 'commit_id.short() ++ " " ++ description.first_line() ++ "\n"'
```
Expected: the merge change's commit id and description.

- [ ] **Step 3: Push**

Run:
```bash
jj git push --bookmark claude2boogaloo
```
Expected: `Move forward bookmark claude2boogaloo from b876a2be to <merge sha>`. This is a fast-forward on origin; no force.

- [ ] **Step 4: Start the next change so the merge is not edited further**

Run:
```bash
jj new
```

---

## Out of scope (do not do in this merge)

- Updating `origin/main`.
- Adding a `docs/plans/` "Done" entry to `TODO-misc.md`. Nothing in that file tracks the upstream sync.
- Any change under `crates/`.
- Editing `deny.toml`, the `Dockerfile`, or `container.yml`, even if `just ci` or the Container workflow later complains. Report instead.
