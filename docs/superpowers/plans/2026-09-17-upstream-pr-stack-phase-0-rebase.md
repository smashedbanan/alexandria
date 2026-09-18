# Upstream PR Stack, Phase 0: Rebase onto main Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebase ten of the eleven open draft PRs (`pr/s1`…`pr/s8`, `pr/a`, `pr/c`) onto upstream `main` at `e251a5e`, prove nothing was lost, push them, and tell the reviewer.

**Architecture:** One `jj rebase` moves all 60 commits; conflicts are resolved bottom-up by editing the conflicted change so jj propagates the fix upward. Verification is `just ci` at every tip from a scratch jj workspace, plus a file-set check proving the rebase only changed files that upstream also changed. `pr/b` (PR #18) is deliberately left on the old base.

**Tech Stack:** jj 0.45 (colocated with git), `just`, `gh`, cargo stable, node 26.

**Spec:** `docs/superpowers/specs/2026-09-17-upstream-pr-stack-rebase-and-review-fixes-design.md` (read "Mechanics → Phase 0" and "Background" before starting).

## Global Constraints

- Repo: `/home/derek/mnt/evo_ssd/PROJECTS/repos/git/alexandria`. Remotes: `origin` = `smashedbanan/alexandria` (fork, push target), `upstream` = `cebarks/alexandria` (PR target).
- Destination commit: `main@upstream` = change `zozrllxzknvv`, commit `e251a5e`. Old base: `2da0cbe`.
- Rebase roots (change ids): `zwmsmwzpytnw` (`a3c375e`, root of `pr/s1`), `ltomxynuvtvw` (`2c2eef7`, root of `pr/a`), `qlvuunkxuosl` (`163ca8d`, root of `pr/c`). Do **not** rebase `rmtkvumtqmxl` (root of `pr/b`).
- The ten bookmarks, in order: `pr/s1-deps-tooling pr/s2-embedding-config pr/s3-sessions pr/s4-hnsw pr/s5-bench pr/s6-repo-boundary-audit pr/s7-256-tokens pr/s8-access-dedup pr/a-claude-hooks pr/c-docs`.
- Never `jj new`, `jj describe`, `jj squash` without `-m`. Never `jj resolve --tool :ours` / `:theirs` (per-file, silently drops fork-only hunks).
- jj signs commits with an ssh key (`signing.behavior = "own"`). If a jj command fails inside the sandbox with a signing error, re-run it outside the sandbox. Do not disable signing.
- Loop over conflicted commits by **change id**, never commit id (commit ids change on every rewrite).
- The `claude2boogaloo` branch and the `wip` stack on top of it are not part of this work. `jj log -r '(2da0cbe..pr/s8-access-dedup) & ::@'` must stay empty.
- Scratch files live in `/tmp/pr-rebase/`. Anything a later phase needs is written into the PR comment, not left in `/tmp`.
- Build gate before any `cargo build`/`check`: `cargo fmt --all -- --check` then `cargo clippy --workspace --all-targets --all-features -- -D warnings`. `just ci` runs both.
- Run cargo on **stable**.

---

### Task 1: Record the pre-rebase state and park the pi series

**Files:**
- Create: `/tmp/pr-rebase/tips-before.txt`
- jj bookmark: `parked/pi-old-series`

**Interfaces:**
- Produces: `/tmp/pr-rebase/tips-before.txt` with lines `<bookmark> <commit-id>` for all 11 bookmarks; Task 5 reads it.

- [ ] **Step 1: Confirm nothing is dirty and the bookmarks match origin**

Run:
```bash
cd /home/derek/mnt/evo_ssd/PROJECTS/repos/git/alexandria
jj st --no-pager | head -3
jj bookmark list -a | grep -E '^pr/|^  @origin' | paste - - | awk '{ if ($2 != $5) print "DIVERGED", $0 }'
```
Expected: working copy `(empty) wip`; the awk prints nothing (every `pr/*` local commit id equals its `@origin` id).

- [ ] **Step 2: Confirm upstream main is still `e251a5e`**

Run:
```bash
jj git fetch --remote upstream
jj log -r 'main@upstream' --no-graph -T 'change_id.short() ++ " " ++ commit_id.short() ++ "\n"'
```
Expected: `zozrllxzknvv e251a5ed88e2`. If it moved, stop and report the new head; the spec's numbers are pinned to `e251a5e`.

- [ ] **Step 3: Record all eleven tips**

Run:
```bash
mkdir -p /tmp/pr-rebase
for b in pr/s1-deps-tooling pr/s2-embedding-config pr/s3-sessions pr/s4-hnsw pr/s5-bench pr/s6-repo-boundary-audit pr/s7-256-tokens pr/s8-access-dedup pr/a-claude-hooks pr/b-pi-extension pr/c-docs; do
  printf "%s %s\n" "$b" "$(jj log -r "$b" --no-graph -T 'commit_id')"
done | tee /tmp/pr-rebase/tips-before.txt
wc -l /tmp/pr-rebase/tips-before.txt
```
Expected: 11 lines, `pr/s8-access-dedup 0ba0d606…`, `pr/a-claude-hooks 821de452…`, `pr/c-docs d86435b2…`, `pr/b-pi-extension 8bdcbbbf…`.

- [ ] **Step 4: Park the old pi series**

Run:
```bash
jj bookmark create parked/pi-old-series -r pr/b-pi-extension
jj bookmark list | grep parked
```
Expected: `parked/pi-old-series: upttwknx 8bdcbbbf …`. This bookmark is never pushed.

---

### Task 2: Rebase the 60 commits in one operation

**Files:**
- Create: `/tmp/pr-rebase/conflicts-after-rebase.txt`

**Interfaces:**
- Produces: the ten bookmarks now sit on descendants of `zozrllxzknvv`; `/tmp/pr-rebase/conflicts-after-rebase.txt` lists conflicted change ids for Task 3.

- [ ] **Step 1: Count what will move**

Run:
```bash
jj log -r 'descendants(zwmsmwzpytnw | ltomxynuvtvw | qlvuunkxuosl)' --no-graph -T '"x"' | wc -c
```
Expected: `60`. If it is 71, `pr/b` is a descendant of one of the roots and the revset is wrong; stop.

- [ ] **Step 2: Rebase**

Run:
```bash
jj rebase -s zwmsmwzpytnw -s ltomxynuvtvw -s qlvuunkxuosl -d zozrllxzknvv 2>&1 | tail -5
```
Expected: `Rebased 60 commits onto destination` followed by `New conflicts appeared in N commits` and a hint. jj never aborts a rebase on conflicts.

- [ ] **Step 3: Confirm the bookmarks moved and the wip stack did not**

Run:
```bash
jj log -r 'zozrllxzknvv..(pr/s8-access-dedup | pr/a-claude-hooks | pr/c-docs)' --no-graph -T '"x"' | wc -c
jj log -r '(2da0cbe..pr/s8-access-dedup) & ::@' --no-graph -T 'commit_id.short() ++ "\n"' | wc -l
jj log -r 'pr/b-pi-extension' --no-graph -T 'commit_id.short() ++ "\n"'
```
Expected: `60`, `0`, `8bdcbbbf` (pr/b untouched).

- [ ] **Step 4: List conflicted commits bottom-up**

Run:
```bash
jj log -r 'conflicts() & zozrllxzknvv..' --reversed --no-graph \
  -T 'change_id.short() ++ " " ++ bookmarks ++ " " ++ description.first_line() ++ "\n"' \
  | tee /tmp/pr-rebase/conflicts-after-rebase.txt
wc -l /tmp/pr-rebase/conflicts-after-rebase.txt
```
Expected: a list, oldest first. The dry-run merge-tree counts predict conflicts concentrated in `pr/s1` (Cargo.lock, Cargo.toml, justfile, AGENTS.md), `pr/s3` (session_repo.rs, server.rs, AGENTS.md), `pr/s6` (memory_repo.rs, server.rs), `pr/s8` (server.rs, docs). `pr/a` and `pr/c` should be clean.

---

### Task 3: Resolve conflicts bottom-up

**Files:**
- Modify: whichever files `jj resolve --list` reports, one conflicted change at a time.

**Interfaces:**
- Consumes: `/tmp/pr-rebase/conflicts-after-rebase.txt`.
- Produces: `jj log -r 'conflicts()'` empty.

**Resolution rules (from the #19 review and the spec):** stack facts win on *state* (v006 is not the head any more, `main` has v007; 9 tools becomes 9 + reminders tools; 0.10 floor), `main`'s additions win on *structure* (reminders sections, `ext-test`/`verify-assets` recipes, debug UI handlers, CSRF guard). Keep both sides' new functions. When `main` and the stack both edit the same sentence, keep `main`'s wording and re-apply only the fact the stack commit was adding.

Per-file guidance:

| file | rule |
|---|---|
| `Cargo.lock` | never hand-merge. `jj restore --from zozrllxzknvv Cargo.lock` then `cargo metadata --format-version 1 >/dev/null` regenerates the minimal delta for that commit's `Cargo.toml`. |
| `Cargo.toml`, `crates/*/Cargo.toml` | keep `main`'s new dependencies as `main` wrote them (do not add `default-features = false` to deps the stack never saw; the `pr/s1` PR comment will say so). Keep the stack's feature lists on deps both sides have. |
| `justfile` | keep `main`'s `ext-test`, `ext-install`, `test-all`, `vendor-assets`, `verify-assets`; keep the stack's `--workspace` flags; `ci: fmt lint test ext-test deny verify-assets`. Drop nothing of `main`'s. |
| `AGENTS.md` | keep `main`'s new bullets (reminders, ext-test, `DEFAULT_MIN_SIMILARITY`, "recount rather than trusting the number"). Apply the stack bullet's *fact* only where `main` lacks it. If `main` already states it, take `main`'s text and let the stack hunk vanish. |
| `crates/alexandria-mcp/src/server.rs` | additive on both sides; keep `main`'s reminders and debug handlers and the stack's new methods. The stack's removal of an inline query only applies to queries that still exist in `main`'s text. |
| `memory_repo.rs`, `session_repo.rs` | additive; keep both. Re-run `cargo fmt --all` afterwards. |
| `crates/alexandria/src/config.rs`, `main.rs` | additive; `main`'s reminders config and the stack's embedding keys coexist. |
| `docs/configuration.md`, `docs/roadmap.md`, `docs/session-memory.md` | keep both sides' sections; prefer `main`'s ordering. |

- [ ] **Step 1: Take the oldest conflicted change and edit it**

Run (replace `CHANGE` with the first change id in the file):
```bash
jj edit CHANGE
jj resolve --list
```
Expected: the list of conflicted paths for this commit. `jj st` shows `Working copy` at that change with conflict markers on disk.

- [ ] **Step 2: Resolve each listed file**

For `Cargo.lock`:
```bash
jj restore --from zozrllxzknvv Cargo.lock
cargo metadata --format-version 1 >/dev/null
jj diff --stat -- Cargo.lock
```
Expected: `Cargo.lock` no longer conflicted; the diff is the minimal delta for this commit's manifest.

For every other file, open it and resolve the jj conflict markers by hand. jj's markers look like:
```
<<<<<<< Conflict 1 of 2
%%%%%%% Changes from base to side #1        <- a *diff* (lines prefixed -/+), not a full copy
-old line
+main's line
+++++++ Contents of side #2                 <- the stack's full version of the region
stack's line
>>>>>>> Conflict 1 of 2 ends
```
Apply the `%%%%%%%` diff onto the `+++++++` contents by hand, following the per-file table, and delete the markers. For `.rs` files run `cargo fmt --all` after editing.

- [ ] **Step 3: Confirm this change is resolved and see what auto-resolved above it**

Run:
```bash
jj resolve --list 2>&1 | head -3
jj log -r 'conflicts() & zozrllxzknvv..' --reversed --no-graph -T 'change_id.short() ++ " " ++ description.first_line() ++ "\n"'
```
Expected: the first command reports no conflicts for this revision (jj prints an error like `No conflicts found at this revision`, which is the success signal here); the second list is shorter than before. Descendants that only conflicted because of this file are gone from it.

- [ ] **Step 4: Repeat Steps 1–3 for the next oldest change until the list is empty**

Fallback from the spec (Mechanics step 3): if one layer has five or more conflicted commits that all touch the same file, squash that layer to one commit before resolving, so the file is resolved once: `jj squash --from '<layer root>::<layer tip> ~ <layer tip>' --into <layer tip> -m "<the tip's description>"` (uses `-m`; the layer's bookmark stays on the tip). Do not do this to `pr/s3` (#11) or `pr/s6` (#14), which the reviewer accepted as shaped.

Run when done:
```bash
jj log -r 'conflicts()' --no-graph -T 'change_id.short() ++ "\n"' | wc -l
jj log -r '(2da0cbe..pr/s8-access-dedup) & ::@' --no-graph | wc -l
```
Expected: `0` and `0`.

- [ ] **Step 5: Drop any commit the rebase emptied**

A stack commit can become empty when `main` already contains its change (candidates: the `pr/s1` tip "bump rmcp 3.2 -> 3.3 and smallvec" after `main`'s `cargo update`; AGENTS.md-only doc commits whose fact `main` now states). Run:
```bash
jj log -r 'empty() & zozrllxzknvv.. & ~merges()' --no-graph -T 'change_id.short() ++ " " ++ bookmarks ++ " " ++ description.first_line() ++ "\n"'
```
For each listed change: `jj abandon CHANGE`. jj reparents its children and moves any bookmark on it to the parent. Re-run the query; expected empty. Record which commits were abandoned in `/tmp/pr-rebase/abandoned.txt` (one line each: old description) — the #9 comment in Task 6 lists them.

- [ ] **Step 6: Return to the wip working copy**

Run:
```bash
jj edit vxynnnyt
jj log -r '@' --no-graph -T 'description.first_line() ++ "\n"'
```
Expected: `wip`. (`vxynnnyt` is the empty wip created when the spec amendment was committed; if `jj log -r 'description(exact:"wip") & @::'` shows a different id, use that one.)

---

### Task 4: `just ci` at every tip from a scratch workspace

**Files:**
- Create: `/tmp/pr-rebase/ci-<bookmark>.log` (ten files), `/tmp/pr-rebase/ci-summary.txt`
- jj workspace: `/home/derek/mnt/evo_ssd/PROJECTS/repos/git/alexandria-ci`

**Interfaces:**
- Produces: `/tmp/pr-rebase/ci-summary.txt` with one `PASS`/`FAIL` line per bookmark; Task 6's push is gated on ten `PASS` lines.

- [ ] **Step 1: Create the scratch workspace and share the target directory**

Run:
```bash
cd /home/derek/mnt/evo_ssd/PROJECTS/repos/git/alexandria
jj workspace add ../alexandria-ci
cd ../alexandria-ci
jj describe -r @ -m "wip"
export CARGO_TARGET_DIR=/home/derek/mnt/evo_ssd/PROJECTS/repos/git/alexandria/target
jj workspace list
```
Expected: two workspaces, `default` and `alexandria-ci`.

- [ ] **Step 2: Install the pi companion's dev dependencies once**

`just ci` on every rebased tip now includes `ext-test` (inherited from `main`'s justfile), which needs `node_modules`. Run at the first tip:
```bash
jj new pr/s1-deps-tooling -m "scratch: ci pr/s1-deps-tooling"
just ext-install 2>&1 | tail -3
```
Expected: `npm ci` completes; `contrib/pi/extensions/alexandria/node_modules/` exists (gitignored, so `jj st` shows nothing).

- [ ] **Step 3: Run `just ci` at each tip in stack order**

Each iteration starts a fresh scratch child of the tip with `jj new` and leaves it in place; the scratch commits are abandoned together in Step 5. Never `jj abandon @` inside the loop: that would move the workspace onto the PR tip itself, and any stray file would then be snapshotted into the PR.

Run:
```bash
: > /tmp/pr-rebase/ci-summary.txt
for b in pr/s1-deps-tooling pr/s2-embedding-config pr/s3-sessions pr/s4-hnsw pr/s5-bench pr/s6-repo-boundary-audit pr/s7-256-tokens pr/s8-access-dedup pr/a-claude-hooks pr/c-docs; do
  jj new "$b" -m "scratch: ci $b" >/dev/null 2>&1
  if just ci > "/tmp/pr-rebase/ci-$(basename "$b").log" 2>&1; then r=PASS; else r=FAIL; fi
  printf "%-28s %s\n" "$b" "$r" | tee -a /tmp/pr-rebase/ci-summary.txt
  jj diff --stat | grep -q . && echo "WARNING: $b left tracked changes: $(jj diff --stat | tail -1)"
done
cat /tmp/pr-rebase/ci-summary.txt
```
Expected: ten `PASS` lines and no `WARNING`. Each run is fmt + clippy + Rust tests + ext-test + cargo-deny + verify-assets; the first takes several minutes (cold clippy on the shared target dir), later ones less. The huggingface cache is warm (`~/.cache/huggingface/hub/models--sentence-transformers--all-MiniLM-L6-v2` exists) so the #9 download race cannot fire here; it is Phase 1's job.

- [ ] **Step 4: On any FAIL, fix it in the layer commit and re-run from there**

Read `/tmp/pr-rebase/ci-<bookmark>.log`. If the failure is a conflict-resolution mistake (a duplicated function, a missing import, a doc test referencing a renamed recipe), fix it in the **default** workspace by `jj edit`-ing the commit that introduced the resolution (find it with `jj log -r 'zozrllxzknvv..<bookmark>' -p -- <file>`), then in the ci workspace re-run Step 3's loop body for that bookmark and every bookmark above it. If the failure pre-dates the rebase (reproduce at the old commit id from `/tmp/pr-rebase/tips-before.txt` with `jj new <old-id>` in the ci workspace), record it in `/tmp/pr-rebase/ci-summary.txt` as `FAIL (pre-existing: <one line>)` and continue; it is not this phase's job.

- [ ] **Step 5: Abandon the scratch commits and remove the workspace**

Run (still in the ci workspace; move `@` onto `main` first so no scratch commit is the working copy when abandoned):
```bash
jj new zozrllxzknvv -m "wip"
jj abandon -r 'description(glob:"scratch: ci *")' 2>&1 | tail -2
cd /home/derek/mnt/evo_ssd/PROJECTS/repos/git/alexandria
jj workspace forget alexandria-ci
rm -rf ../alexandria-ci
jj workspace list
jj log -r 'description(glob:"scratch: ci *")' --no-graph | wc -l
for b in pr/s1-deps-tooling pr/s8-access-dedup pr/a-claude-hooks pr/c-docs; do jj log -r "$b" --no-graph -T 'bookmarks ++ " " ++ commit_id.short() ++ "\n"'; done
```
Expected: `Abandoned 10 commits`; only `default` workspace; `0` scratch commits; the four bookmark commit ids are unchanged from before Task 4 (compare against `jj op log` if in doubt).

---

### Task 5: Prove the rebase changed only what upstream changed

**Files:**
- Create: `/tmp/pr-rebase/octopus-check.txt`, git worktree `/tmp/pr-rebase/wt` (removed at the end)

**Interfaces:**
- Consumes: `/tmp/pr-rebase/tips-before.txt`.
- Produces: `/tmp/pr-rebase/octopus-check.txt`: the list of files whose content differs between the pre- and post-rebase octopus merges, each annotated `upstream-touched` or `UNEXPECTED`.

The check: build the octopus merge of `pr/s8 + pr/a + pr/c` before the rebase (from the recorded old commit ids) and after. Every file that differs between the two must be a file upstream changed in `2da0cbe..e251a5e`. A file that differs but upstream never touched means a conflict resolution altered fork-only content.

- [ ] **Step 1: Build both octopus merges in a throwaway git worktree**

Run:
```bash
cd /home/derek/mnt/evo_ssd/PROJECTS/repos/git/alexandria
OLD_S8=$(awk '$1=="pr/s8-access-dedup"{print $2}' /tmp/pr-rebase/tips-before.txt)
OLD_A=$(awk '$1=="pr/a-claude-hooks"{print $2}' /tmp/pr-rebase/tips-before.txt)
OLD_C=$(awk '$1=="pr/c-docs"{print $2}' /tmp/pr-rebase/tips-before.txt)
NEW_S8=$(jj log -r pr/s8-access-dedup --no-graph -T commit_id)
NEW_A=$(jj log -r pr/a-claude-hooks --no-graph -T commit_id)
NEW_C=$(jj log -r pr/c-docs --no-graph -T commit_id)
git worktree add --detach /tmp/pr-rebase/wt "$OLD_S8"
git -C /tmp/pr-rebase/wt -c user.name=scratch -c user.email=scratch@local merge --no-edit "$OLD_A" "$OLD_C" >/dev/null && PRE=$(git -C /tmp/pr-rebase/wt rev-parse HEAD)
git -C /tmp/pr-rebase/wt checkout --detach "$NEW_S8" >/dev/null 2>&1
git -C /tmp/pr-rebase/wt -c user.name=scratch -c user.email=scratch@local merge --no-edit "$NEW_A" "$NEW_C" >/dev/null && POST=$(git -C /tmp/pr-rebase/wt rev-parse HEAD)
echo "PRE=$PRE POST=$POST"
```
Expected: both merges complete without `CONFLICT` (the #19 review measured zero textual conflicts between `pr/c` and every stack ref; `pr/a` is `contrib/claude` only). If the **post** merge conflicts, two PRs now resolve `main`'s changes differently; resolve it consistently in the offending layer commit (Task 3 procedure) and redo this step.

- [ ] **Step 2: Diff the file sets**

Run:
```bash
git diff --name-only "$PRE" "$POST" | sort > /tmp/pr-rebase/changed-by-rebase.txt
git diff --name-only 2da0cbe e251a5e | sort > /tmp/pr-rebase/changed-by-upstream.txt
comm -23 /tmp/pr-rebase/changed-by-rebase.txt /tmp/pr-rebase/changed-by-upstream.txt > /tmp/pr-rebase/unexpected.txt
{ comm -12 /tmp/pr-rebase/changed-by-rebase.txt /tmp/pr-rebase/changed-by-upstream.txt | sed 's/$/  upstream-touched/'; sed 's/$/  UNEXPECTED/' /tmp/pr-rebase/unexpected.txt; } > /tmp/pr-rebase/octopus-check.txt
wc -l /tmp/pr-rebase/unexpected.txt; cat /tmp/pr-rebase/unexpected.txt
```
Expected: `0` unexpected files. `Cargo.lock` is upstream-touched (`7487c1c`), so it is allowed to differ.

- [ ] **Step 3: Eyeball the upstream-touched files that differ**

For each file in `octopus-check.txt` marked `upstream-touched`, the diff between the two octopi must be explainable as "upstream's change to this file" plus nothing else:
```bash
for f in $(comm -12 /tmp/pr-rebase/changed-by-rebase.txt /tmp/pr-rebase/changed-by-upstream.txt); do
  echo "=== $f: rebase-delta $(git diff --numstat "$PRE" "$POST" -- "$f" | cut -f1,2 | tr '\t' '/')  upstream-delta $(git diff --numstat 2da0cbe e251a5e -- "$f" | cut -f1,2 | tr '\t' '/')"
done
```
A file whose rebase-delta is much larger than its upstream-delta lost or duplicated fork content; open `git diff "$PRE" "$POST" -- <file>` and compare against `git diff 2da0cbe e251a5e -- <file>`. Fix in the layer commit and redo Task 5 from Step 1 (the post octopus must be rebuilt). If any `UNEXPECTED` file survives with a justified reason (e.g. `cargo fmt` reflowed a stack-only file), write the reason next to it in `octopus-check.txt`; it goes into the #9 comment.

- [ ] **Step 4: Remove the worktree**

Run:
```bash
git worktree remove --force /tmp/pr-rebase/wt
git worktree prune
git worktree list
```
Expected: only the main worktree.

---

### Task 6: Push the ten bookmarks and comment on #9 and #18

**Files:**
- Create: `/tmp/pr-rebase/comment-9.md`, `/tmp/pr-rebase/comment-18.md`

**Interfaces:**
- Consumes: `/tmp/pr-rebase/ci-summary.txt` (ten `PASS`), `/tmp/pr-rebase/octopus-check.txt`, `/tmp/pr-rebase/abandoned.txt`.

- [ ] **Step 1: Gate**

Run:
```bash
grep -c '^pr/.* PASS$' /tmp/pr-rebase/ci-summary.txt
wc -l < /tmp/pr-rebase/unexpected.txt
jj log -r 'conflicts()' --no-graph | wc -l
jj log -r 'zozrllxzknvv..(pr/s8-access-dedup | pr/a-claude-hooks | pr/c-docs) & description(exact:"wip")' --no-graph | wc -l
```
Expected: `10`, `0` (or every remaining line annotated in `octopus-check.txt`), `0`, `0`. Do not push on any other result.

- [ ] **Step 2: Push**

Run:
```bash
jj git push --remote origin \
  -b pr/s1-deps-tooling -b pr/s2-embedding-config -b pr/s3-sessions -b pr/s4-hnsw \
  -b pr/s5-bench -b pr/s6-repo-boundary-audit -b pr/s7-256-tokens -b pr/s8-access-dedup \
  -b pr/a-claude-hooks -b pr/c-docs 2>&1 | tail -12
```
Expected: ten `Move forward bookmark`/`Move sideways bookmark` lines (sideways is normal after a rebase) and no `Refusing`. If jj refuses because a remote bookmark is "conflicted" or "untracked", run `jj bookmark track <name>@origin` for it and retry; do not use `--allow-new` unless the bookmark is genuinely missing on origin.

- [ ] **Step 3: Confirm GitHub sees the new heads**

Run:
```bash
for n in 9 10 11 12 13 14 15 16 17 19; do
  gh pr view $n -R cebarks/alexandria --json number,headRefOid,mergeable,baseRefName --jq '"\(.number) \(.headRefOid[:12]) \(.mergeable) base=\(.baseRefName)"'
done
jj log -r 'pr/s1-deps-tooling | pr/a-claude-hooks | pr/c-docs' --no-graph -T 'bookmarks ++ " " ++ commit_id.short(12) ++ "\n"'
```
Expected: each PR's `headRefOid` equals the local bookmark commit; `mergeable` is `MERGEABLE` or `UNKNOWN` (GitHub recomputes lazily), never `CONFLICTING`; base is `main`.

- [ ] **Step 4: Write and post the #9 comment**

Write `/tmp/pr-rebase/comment-9.md` from this template, filling the three bracketed lists from the scratch files:
```markdown
## Rebased onto `e251a5e`

The whole series (#9–#17, #19) now sits on current `main`. Per-PR fixes for the 2026-09-16 review round follow in stack order, one PR at a time; I'll ping when the stack is ready for a second pass. Nothing else has changed yet.

**Verification:** `just ci` (fmt, clippy `-D warnings`, tests, `ext-test`, `cargo deny`, `verify-assets`) passes at every one of the ten tips. An octopus merge of `pr/s8 + pr/a + pr/c` before and after the rebase differs only in files that `2da0cbe..e251a5e` also changed.

**Conflict resolutions worth knowing about:**
- `justfile`: kept `main`'s `ext-test`/`ext-install`/`verify-assets` recipes and `ci` line; the stack's `--workspace` flags apply on top.
- `Cargo.toml`: dependencies `main` added after the fork (reminders) keep `main`'s feature spelling; the `default-features = false` sweep in #9 only covers dependencies the fork saw.
- `AGENTS.md`: where `main` already states a fact the stack was adding (v006 head, `DEFAULT_MIN_SIMILARITY`, "recount rather than trust the number"), `main`'s wording stands.
- [one line per file from octopus-check.txt that needed a judgement call]

**Commits the rebase emptied (dropped):** [lines from abandoned.txt, or "none"]

**Not rebased:** #18. `main` relocated the pi extension (`alexandria-auto-recall/` → `alexandria/`) and rewrote `index.ts`/`config.ts`/`mcp-client.ts` for reminders, so it will be rebuilt on the new layout as the redo you asked for, not rebased.
```
Then:
```bash
gh pr comment 9 -R cebarks/alexandria --body-file /tmp/pr-rebase/comment-9.md
```
Expected: a comment URL.

- [ ] **Step 5: Write and post the #18 comment**

`/tmp/pr-rebase/comment-18.md`:
```markdown
Not rebased with the rest of the series: `main` moved `contrib/pi/extensions/alexandria-auto-recall/` to `contrib/pi/extensions/alexandria/` and rewrote `index.ts`, `config.ts` and `mcp-client.ts` for reminders, so a rebase is modify/delete on every commit. This PR will be force-pushed as a fresh series on the new layout, carrying the redo you asked for (trigger-token captures, no negation stripping, bare `don't`, verb-bearing corrections, the round-trip test) plus the detector fixes and tests you listed as good, ported into `tests/*.test.ts` under `ext-test`. `test-pi`/`typecheck-pi`/`pi-tests` go away in favour of `ext-test` and the "Pi companion" job. Finalize-at-shutdown comes back separately with its own design.
```
```bash
gh pr comment 18 -R cebarks/alexandria --body-file /tmp/pr-rebase/comment-18.md
```
Expected: a comment URL.

- [ ] **Step 6: Record the outcome for later phases**

Store one memory in Alexandria (`mcp__alexandria__store_memory`) with content:
`Alexandria upstream PR stack rebased onto cebarks/alexandria main e251a5e on <date>: pr/s1..s8, pr/a, pr/c pushed; pr/b (#18) deliberately left on 2da0cbe and parked as parked/pi-old-series because main relocated the pi extension to contrib/pi/extensions/alexandria/. Phase 0 of docs/superpowers/specs/2026-09-17-upstream-pr-stack-rebase-and-review-fixes-design.md is done; next is Phase 1 (#9 hub.rs race fix).`
tags: `alexandria`, `upstream-pr`, `pr-stack-2026-09`.

Then tick every box in this file, commit the plan update:
```bash
jj describe -r @ -m "docs(plan): mark Phase 0 of the upstream PR-stack rebase done"
jj new -m "wip"
```

---

## Plan index for the remaining phases

Each phase below gets its own plan file, written **after** Phase 0 lands, because its code steps
must be written against the rebased tree. File names are fixed now so they can be referenced.

| phase | PR | bookmark | plan file | spec section |
|---|---|---|---|---|
| 1 | #9 | `pr/s1-deps-tooling` | `2026-09-17-upstream-pr-stack-phase-1-pr9-hub-race.md` | "#9" |
| 2 | #10 | `pr/s2-embedding-config` | `…-phase-2-pr10-batch-size.md` | "#10" |
| 3 | #11 | `pr/s3-sessions` | `…-phase-3-pr11-sessions.md` | "#11" |
| 4 | #12 | `pr/s4-hnsw` | `…-phase-4-pr12-hnsw-operator.md` | "#12" |
| 5 | #13 | `pr/s5-bench` | `…-phase-5-pr13-bench-claims.md` | "#13" |
| 6 | #14 | `pr/s6-repo-boundary-audit` | `…-phase-6-pr14-audit-docs.md` | "#14" |
| 7 | #15 | `pr/s7-256-tokens` | `…-phase-7-pr15-token-lock.md` | "#15" |
| 8 | #16 | `pr/s8-access-dedup` → renamed | `…-phase-8-pr16-reshape.md` | "#16" |
| 9 | #17 | `pr/a-claude-hooks` | `…-phase-9-pr17-tool-error-gate.md` | "#17" |
| 10 | #18 | `pr/b-pi-extension` (rebuilt) | `…-phase-10-pr18-rebuild.md` | "#18" |
| 11 | #19 | `pr/c-docs` | `…-phase-11-pr19-docs.md` | "#19" |

Phases 1–8 are strictly sequential (each inserts commits into a layer the next one sits on).
Phases 9, 10 and 11 depend only on Phase 0, and Phase 11 is pushed last per the reviewer's hold.
