# Debug Memories: Access Count, Last Touched, Sortable Columns

**Date:** 2026-09-14
**Status:** Approved design, pre-implementation. Amended 2026-09-14 after review, with approval:
`bench-retrieval`'s corpus read moves to `MemoryRepo::live_facts`.

## Goal

The `/debug/memories` table shows each fact's **Access Count** and **Last Touched**, and every column
header sorts the whole result set, not just the visible page.

## Background

- The counters live on `heat_state` (`memory`, `access_count`, `last_touched`), a separate table
  from `fact`, one row per fact created through `store_memory`/`import_document`. Facts created any
  other way (tests, `update_memory` lineage snapshots) may have no `heat_state` row.
- `access_count` is bumped by `record_access` (top N of `retrieve_memories`). `last_touched` is also
  bumped by spreading activation (`HeatRepo::add_heat`), so the column is labelled **Last Touched**,
  not "Last Accessed". It also defaults to the heat row's creation time (`DEFAULT time::now()` in
  `v001`), so a never-accessed fact that has a heat row shows about its Created time, not `—`.
- `/debug/memories` paginates in the database (`LIMIT`/`START`, 50 per page). The live corpus on
  2026-09-14 had 2,105 facts (1,369 active, 736 deleted). A browser-side sort would only reorder the
  50 visible rows, so sorting is server-side.
- The detail page (`/debug/memories/{id}`) already shows both values; it does not change.
- `MemoryRepo::list` has a second caller. `bench-retrieval` (`crates/alexandria/src/bench.rs`,
  `run()`) reads the whole live corpus through it, newest first and capped at `CORPUS_CAP`, and uses
  each row's `id`, `embedding` and `created_at`.

## Scope

In: the `/debug/memories` list table. Also moving `bench-retrieval`'s corpus read onto its own
repo method, because the table's new row type drops `embedding`. The bench's query and results stay
the same.

Out: every other debug page (dashboard, memory detail, clusters, graph, maintenance log, query
tester); any change to heat write paths; any schema migration.

## Behaviour

### Columns

Order: ID, Content, Tags, Confidence, Created, **Access Count**, **Last Touched**.

- Access Count renders the integer; Last Touched renders `YYYY-MM-DD HH:MM UTC` (same format as
  Created).
- A fact without a `heat_state` row renders `—` in both cells.

### Sorting

- Every header is a link. URL params: `sort` ∈ {`id`, `content`, `tags`, `confidence`, `created`,
  `access_count`, `last_touched`} and `dir` ∈ {`asc`, `desc`}.
- Clicking an inactive header sorts that column descending. Clicking the active header flips the
  direction. The active header shows ` ▼` (desc) or ` ▲` (asc).
- `sort` and `dir` parse independently. A missing or unrecognised `sort` means Created. A missing
  `dir`, or any value other than `asc`, means descending. With neither param the order is Created
  descending, which is today's order. `?sort=bogus&dir=asc` gives Created ascending.
- Header links reset `offset` to 0 and keep `search`, `tag`, `include_deleted`, `limit`.
- Prev/Next links carry `sort` and `dir`. The htmx filter form carries them as hidden inputs, so
  typing a search keeps the sort.
- `id ASC` breaks ties for every column except ID itself, so pages neither repeat nor skip rows when
  many facts share a value (most access counts are 0).
- Facts without a `heat_state` row sort as the lowest value: first ascending, last descending.
  This is SurrealDB's `NONE` ordering, verified in the spike.

### Safety

The `sort` param never reaches SQL text. The handler maps it through a fixed table to a
`SortColumn` enum, and the enum maps to a fixed field name. `dir` only selects between two
literals.

## Design

### Storage (`alexandria-storage`)

- `models::FactListRow` — the list row: `id`, `content`, `confidence`, `tags`, `created_at`,
  `deleted`, `access_count: Option<i64>`, `last_touched: Option<DateTime<Utc>>`. It omits
  `embedding` and `metadata`, which the table never shows.
- `repos::SortColumn` (enum, one variant per column) and `repos::FactSort { column, descending }`,
  `Default` = Created descending.
- `MemoryRepo::list(search, tag, include_deleted, sort: FactSort, limit, offset) -> Vec<FactListRow>`.
  `count()` is unchanged. After this change the debug table is `list`'s only caller.
- `MemoryRepo::live_facts(limit) -> Vec<Fact>` runs the query `list` ran for the bench, minus
  `START 0`: `SELECT * FROM fact WHERE deleted = false ORDER BY created_at DESC LIMIT $limit`.
  `bench.rs` calls it, and the `CORPUS_CAP` doc comment names it instead of `list`.

Query shape (verified against SurrealDB 3.2.4, in-memory, 2,100 facts with heat rows):

```sql
LET $ac = object::from_entries(SELECT VALUE [<string> memory, access_count] FROM heat_state);
LET $lt = object::from_entries(SELECT VALUE [<string> memory, last_touched] FROM heat_state);
SELECT id, content, confidence, tags, created_at, deleted,
       $ac[<string> id] AS access_count, $lt[<string> id] AS last_touched
FROM fact <where> ORDER BY <field> <DIR>, id ASC LIMIT $limit START $offset
```

Rows are statement index 2.

#### Rejected alternatives (measured in the spike)

| Approach | Time | Why rejected |
|---|---|---|
| Correlated subquery `(SELECT VALUE access_count FROM heat_state WHERE memory = $parent.id)[0]` | ~8 s | Scans `heat_state` once per fact |
| Same plus `DEFINE INDEX … ON heat_state FIELDS memory` | ~8 s | The planner uses the index for `memory = fact:x`, not for `$parent.id` (EXPLAIN: `TableScan`, "unsupported predicate") |
| Same plus `WITH INDEX` hint | ~7.9 s | Same |
| One subquery aliased `heat`, `ORDER BY heat.access_count` | — | Parse error: "Missing order idiom `heat.access_count` in statement selection" |
| **Two `LET` lookup objects** | **~180 ms** | **Chosen** — no schema change, sort and pagination stay in the DB |
| Two flat queries, join and sort in Rust | ~11 ms + Rust sort | Moves filtering, sorting and paging out of the DB; more code for a debug page |

The chosen query rebuilds both lookups on every page load, which is linear in `heat_state`. It
carries a `ponytail:` comment naming the upgrade path: move the counters onto `fact` if the page
gets slow. The timings are in-memory; the on-disk `surrealkv` store will be slower by an unmeasured
factor.

### Debug handler (`alexandria-mcp/src/debug/memories.rs`)

- One `COLUMNS` table of `(url key, header label, SortColumn)` drives both parsing and header
  rendering.
- `memories_url` takes a `FactSort` and appends `sort`/`dir`.
- Two new cells per row.

### Docs

README "Debug Web UI" → Memories bullet mentions sortable columns, access count and last touched.
AGENTS.md: update only the "Current suite: N tests" count under Testing, which feature commits keep
current. Nothing else there changes: no migration, no new tool, and heat is still unused for ranking.

## Testing

Storage (`memory_repo.rs`):

- Sort by access count descending → `hot(7), cold(1), bare(None)`; ascending → `bare, cold, hot`.
  `last_touched` is `Some` for a heat row and `None` without one.
- Sort by ID descending is the reverse of ascending (exercises the no-tie-breaker branch).
- Existing `test_list_and_count_facts` passes with `FactSort::default()`.
- `live_facts` gets no new test. It is one fixed query, the one `list` already ran for the bench, and
  the workspace clippy gate compiles its only caller.

Handler (`memories.rs`):

- Headers `Access Count`/`Last Touched` render. A heat row renders `<td>7</td>`. A fact with no heat
  row ends in `<td>—</td><td>—</td></tr>`.
- `?sort=access_count&dir=desc` puts the hot memory first and shows `Access Count ▼`, and the
  header links to `dir=asc`. `dir=asc` reverses the order and shows `▲`.
- `?sort=content%3BDROP&dir=sideways` returns 200 with `Created ▼`.
- `?sort=access_count&dir=asc&limit=1` → the Next link carries `sort=access_count&amp;dir=asc`.

Gate: `just fmt`, `just lint`, `just test`, all green on stable.
