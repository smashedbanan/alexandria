# Debug Memories Heat Columns + Sorting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Access Count and Last Touched columns to `/debug/memories` and make every column sort the whole result set server-side.

**Architecture:** `MemoryRepo::list` gains a `FactSort` argument and returns `FactListRow`. That row is the fact's listed fields plus `access_count`/`last_touched`, pulled from `heat_state` through two `LET` lookup objects in the same query, so ordering and pagination stay in SurrealDB. The debug handler maps a fixed `COLUMNS` table between URL params and `SortColumn`, renders sortable headers, and carries `sort`/`dir` through pagination and the filter form. `bench-retrieval`, the other `list` caller, needs embeddings, so it moves to a new `MemoryRepo::live_facts` that runs its old query unchanged.

**Tech Stack:** Rust 2024, SurrealDB 3.2.4 (embedded), Axum, htmx 1.9 (already loaded), tokio tests with `tower::ServiceExt::oneshot`.

**Spec:** `docs/superpowers/specs/2026-09-14-debug-memories-heat-columns-design.md`

## Global Constraints

- VCS is **jj**, not git. Always pass `-m`; never run an interactive jj command. Do not `git commit`.
- Build gate before any `cargo build`/`cargo test`, in order: `just fmt` (`cargo fmt --all -- --check`), then `cargo clippy --workspace --all-targets --all-features -- -D warnings`. Use `just fmt-fix` to apply formatting. Avoid `just lint` between test runs: it sets `RUSTFLAGS=-Dwarnings`, which changes the build fingerprint and makes the next plain `cargo test` rebuild every dependency (minutes).
- Run tests on **stable** Rust, not nightly.
- All SurrealDB queries stay in `crates/alexandria-storage`. None in `alexandria-mcp`.
- Request text never reaches SQL. `sort` maps through `COLUMNS` → `SortColumn` → a fixed field name; `dir` picks between the literals `ASC`/`DESC`.
- SurrealDB 3.2: with an explicit projection, every `ORDER BY` idiom must match a projected field or alias exactly. `ORDER BY heat.access_count` with only `heat` projected is a parse error ("Missing order idiom … in statement selection").
- Column label is **Last Touched** (spreading activation bumps it too), never "Last Accessed".
- Scope is `/debug/memories`, plus moving `bench-retrieval`'s corpus read to `MemoryRepo::live_facts` (same query, same results). No schema migration, no changes to the heat write paths, no other debug pages.
- Commit trailer on every change description: `Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>`

---

### Task 1: Storage — sorted `MemoryRepo::list` with heat counters

**Files:**
- Modify: `crates/alexandria-storage/src/models/memory.rs` (add `FactListRow` after `Fact`)
- Modify: `crates/alexandria-storage/src/models/mod.rs:13` (export)
- Modify: `crates/alexandria-storage/src/repos/memory_repo.rs:6-9` (import, new types), `:165-212` (`list`), `:270` (`live_facts`, before `all_ids_and_content`), tests `:319-365` and new test
- Modify: `crates/alexandria-storage/src/repos/mod.rs:10` (export)
- Modify: `crates/alexandria-mcp/src/debug/memories.rs:3` (import) and `:56-63` (keep the workspace compiling; Task 2 finishes the handler)
- Modify: `crates/alexandria/src/bench.rs:144-146` (`CORPUS_CAP` doc comment) and `:543` (corpus read moves to `live_facts`)

**Interfaces:**
- Consumes: `HeatRepo::create_for_memory(&str, f64) -> Result<String>` (returns heat_state id), `HeatRepo::update(id: &str, heat: f64, stability: f64, access_count: i64)`.
- Produces:
  - `alexandria_storage::models::FactListRow { id: Option<RecordId>, content: String, confidence: f64, tags: Vec<String>, created_at: Option<DateTime<Utc>>, deleted: bool, access_count: Option<i64>, last_touched: Option<DateTime<Utc>> }`
  - `alexandria_storage::repos::SortColumn` — `Id | Content | Tags | Confidence | Created | AccessCount | LastTouched`, derives `Debug, Clone, Copy, PartialEq, Eq`
  - `alexandria_storage::repos::FactSort { pub column: SortColumn, pub descending: bool }`, derives `Debug, Clone, Copy, PartialEq, Eq`, `Default` = `{ Created, descending: true }`
  - `MemoryRepo::list(&self, search: Option<&str>, tag: Option<&str>, include_deleted: bool, sort: FactSort, limit: usize, offset: usize) -> Result<Vec<FactListRow>>`
  - `MemoryRepo::live_facts(&self, limit: usize) -> Result<Vec<Fact>>`: live facts, newest first, at most `limit` (bench only)

- [ ] **Step 1: Start the jj change**

```bash
jj new -m "feat(storage): sort MemoryRepo::list and join heat counters

bench-retrieval reads its corpus through the new MemoryRepo::live_facts,
since list's rows no longer carry embeddings.

Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 2: Write the failing storage test**

Add to the `mod tests` block in `crates/alexandria-storage/src/repos/memory_repo.rs`, directly after `test_list_and_count_facts`:

```rust
    #[tokio::test]
    async fn test_list_sorts_by_heat_and_keeps_facts_without_heat() {
        let db = Database::connect_embedded().await.unwrap();
        crate::schema::migrate(db.inner()).await.unwrap();
        let repo = MemoryRepo::new(db.inner());
        let heat = crate::repos::HeatRepo::new(db.inner());

        for (content, accesses) in [("hot", Some(7)), ("cold", Some(1)), ("bare", None)] {
            let id = repo
                .create_fact(content, 0.5, &[0.1, 0.2], &[])
                .await
                .unwrap();
            if let Some(n) = accesses {
                let heat_id = heat.create_for_memory(&id, 1.0).await.unwrap();
                heat.update(&heat_id, 1.0, 1.0, n).await.unwrap();
            }
        }

        let by_access = |descending: bool| FactSort {
            column: SortColumn::AccessCount,
            descending,
        };
        let desc = repo
            .list(None, None, false, by_access(true), 10, 0)
            .await
            .unwrap();
        let got: Vec<_> = desc
            .iter()
            .map(|r| (r.content.as_str(), r.access_count))
            .collect();
        // A fact without a heat_state row sorts as the lowest value.
        assert_eq!(got, vec![("hot", Some(7)), ("cold", Some(1)), ("bare", None)]);
        assert!(desc[0].last_touched.is_some());
        assert!(desc[2].last_touched.is_none());

        let asc = repo
            .list(None, None, false, by_access(false), 10, 0)
            .await
            .unwrap();
        let got: Vec<_> = asc.iter().map(|r| r.content.as_str()).collect();
        assert_eq!(got, vec!["bare", "cold", "hot"]);

        // Id sorts without the `id ASC` tie-breaker; descending must mirror ascending.
        let by_id = |descending: bool| FactSort {
            column: SortColumn::Id,
            descending,
        };
        let ids = |rows: Vec<FactListRow>| -> Vec<String> {
            rows.iter()
                .map(|r| record_id_to_string(r.id.as_ref().unwrap()))
                .collect()
        };
        let asc_ids = ids(repo.list(None, None, false, by_id(false), 10, 0).await.unwrap());
        let mut desc_ids = ids(repo.list(None, None, false, by_id(true), 10, 0).await.unwrap());
        desc_ids.reverse();
        assert_eq!(asc_ids.len(), 3);
        assert_eq!(asc_ids, desc_ids);
    }
```

- [ ] **Step 3: Confirm it fails**

Run: `just fmt-fix && cargo clippy --workspace --all-targets --all-features -- -D warnings`
Expected: compile errors in the `memory_repo.rs` tests (rustc 1.98.1): E0422 `cannot find struct, variant or union type 'FactSort'`, E0425 `cannot find type 'FactListRow'`, E0433 `cannot find type 'SortColumn'`, E0061 `this method takes 5 arguments but 6 arguments were supplied`, and E0609 `no field 'access_count'` / `'last_touched'` on `Fact`.

- [ ] **Step 4: Add `FactListRow`**

In `crates/alexandria-storage/src/models/memory.rs`, insert after the `Fact` struct (after line 15):

```rust

/// One row of `MemoryRepo::list`: the fact fields the debug memories table shows
/// plus its `heat_state` counters, `None` when the fact has no heat_state row.
#[derive(Debug, Clone, Serialize, Deserialize, SurrealValue)]
pub struct FactListRow {
    pub id: Option<RecordId>,
    pub content: String,
    pub confidence: f64,
    pub tags: Vec<String>,
    pub created_at: Option<DateTime<Utc>>,
    pub deleted: bool,
    pub access_count: Option<i64>,
    pub last_touched: Option<DateTime<Utc>>,
}
```

In `crates/alexandria-storage/src/models/mod.rs`, change:

```rust
pub use memory::{Fact, RawRecord};
```

to:

```rust
pub use memory::{Fact, FactListRow, RawRecord};
```

- [ ] **Step 5: Add `SortColumn` and `FactSort`**

In `crates/alexandria-storage/src/repos/memory_repo.rs`, change the model import:

```rust
use crate::models::{Fact, RawRecord};
```

to:

```rust
use crate::models::{Fact, FactListRow, RawRecord};
```

Then insert between `use crate::record_id_to_string;` and `pub struct MemoryRepo<'a> {`:

```rust

/// Sortable columns of the debug memories table. Each maps to a fixed field name,
/// so request text never reaches the query.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SortColumn {
    Id,
    Content,
    Tags,
    Confidence,
    Created,
    AccessCount,
    LastTouched,
}

impl SortColumn {
    fn field(self) -> &'static str {
        match self {
            Self::Id => "id",
            Self::Content => "content",
            Self::Tags => "tags",
            Self::Confidence => "confidence",
            Self::Created => "created_at",
            Self::AccessCount => "access_count",
            Self::LastTouched => "last_touched",
        }
    }
}

/// Order for `MemoryRepo::list`. The default is newest first.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct FactSort {
    pub column: SortColumn,
    pub descending: bool,
}

impl Default for FactSort {
    fn default() -> Self {
        Self {
            column: SortColumn::Created,
            descending: true,
        }
    }
}
```

In `crates/alexandria-storage/src/repos/mod.rs`, change:

```rust
pub use memory_repo::MemoryRepo;
```

to:

```rust
pub use memory_repo::{FactSort, MemoryRepo, SortColumn};
```

- [ ] **Step 6: Replace `MemoryRepo::list`**

Replace the whole existing `list` method, from its doc comment `/// List facts with optional content search, tag filter, and deleted-inclusion.` through its closing `}` just before `/// Count facts matching the same filters as `list``, with:

```rust
    /// List facts with optional content search, tag filter, and deleted-inclusion,
    /// ordered by `sort` with `id` breaking ties so pages neither repeat nor skip rows.
    /// `search` does a case-insensitive substring match against content.
    pub async fn list(
        &self,
        search: Option<&str>,
        tag: Option<&str>,
        include_deleted: bool,
        sort: FactSort,
        limit: usize,
        offset: usize,
    ) -> Result<Vec<FactListRow>> {
        let mut conditions = Vec::new();
        if !include_deleted {
            conditions.push("deleted = false".to_string());
        }
        if search.is_some() {
            conditions
                .push("string::lowercase(content) CONTAINS string::lowercase($search)".to_string());
        }
        if tag.is_some() {
            conditions.push("$tag IN tags".to_string());
        }

        let where_clause = if conditions.is_empty() {
            String::new()
        } else {
            format!("WHERE {}", conditions.join(" AND "))
        };

        let direction = if sort.descending { "DESC" } else { "ASC" };
        let order = match sort.column {
            SortColumn::Id => format!("id {direction}"),
            column => format!("{} {direction}, id ASC", column.field()),
        };

        // A correlated `WHERE memory = $parent.id` subquery scans heat_state once per
        // fact even with an index on `memory` (~8s at 2.1k facts); two lookup objects
        // built up front take ~180ms.
        // ponytail: rebuilds both lookups on every call; move the counters onto `fact` if this page gets slow.
        let query = format!(
            "LET $ac = object::from_entries(SELECT VALUE [<string> memory, access_count] FROM heat_state); \
             LET $lt = object::from_entries(SELECT VALUE [<string> memory, last_touched] FROM heat_state); \
             SELECT id, content, confidence, tags, created_at, deleted, \
             $ac[<string> id] AS access_count, $lt[<string> id] AS last_touched \
             FROM fact {where_clause} ORDER BY {order} LIMIT $limit START $offset"
        );

        let mut q = self
            .db
            .query(&query)
            .bind(("limit", limit as i64))
            .bind(("offset", offset as i64));
        if let Some(s) = search {
            q = q.bind(("search", s.to_string()));
        }
        if let Some(t) = tag {
            q = q.bind(("tag", t.to_string()));
        }

        let mut response = q.await?.check()?;
        // Statements 0 and 1 are the LETs.
        let rows: Vec<FactListRow> = response.take(2)?;
        Ok(rows)
    }
```

- [ ] **Step 7: Update the existing list test's calls**

In `test_list_and_count_facts` (same file), add `FactSort::default()` as the fourth argument to each of the six `repo.list(...)` calls:

```rust
        let all = repo
            .list(None, None, false, FactSort::default(), 10, 0)
            .await
            .unwrap();
```
```rust
        let with_deleted = repo
            .list(None, None, true, FactSort::default(), 10, 0)
            .await
            .unwrap();
```
```rust
        let searched = repo
            .list(Some("alpha"), None, false, FactSort::default(), 10, 0)
            .await
            .unwrap();
```
```rust
        let tagged = repo
            .list(None, Some("tag2"), false, FactSort::default(), 10, 0)
            .await
            .unwrap();
```
```rust
        let page1 = repo
            .list(None, None, false, FactSort::default(), 1, 0)
            .await
            .unwrap();
        let page2 = repo
            .list(None, None, false, FactSort::default(), 1, 1)
            .await
            .unwrap();
```

- [ ] **Step 8: Keep the debug handler compiling**

In `crates/alexandria-mcp/src/debug/memories.rs`, add after `use std::collections::HashMap;` and its blank line:

```rust
use alexandria_storage::repos::FactSort;
```

(rustfmt places it at the top of the second import group, before `use axum::...`.) Then in `list`, change the `repo.list(` call arguments to:

```rust
        .list(
            search.map(|s| s.as_str()),
            tag.map(|s| s.as_str()),
            include_deleted,
            FactSort::default(),
            limit,
            offset,
        )
```

The row loop needs no change: `FactListRow` has the same `id`, `content`, `tags`, `confidence`, `created_at`, `deleted` fields the loop reads.

- [ ] **Step 9: Give `bench-retrieval` its own corpus read**

`bench.rs` `run()` reads `f.embedding` from each `list` row. `FactListRow` has no `embedding`, so without this step the workspace clippy gate fails in `crates/alexandria/src/bench.rs` with E0061 (5 arguments where 6 are needed) and E0609 (no field `embedding` on `FactListRow`).

In `crates/alexandria-storage/src/repos/memory_repo.rs`, insert directly before the doc comment `/// Every fact, deleted ones included, as (id, content). Used by the embedding`:

```rust
    /// Live facts, newest first, at most `limit`. The corpus read for `bench-retrieval`,
    /// which needs the embeddings that `list` rows leave out.
    pub async fn live_facts(&self, limit: usize) -> Result<Vec<Fact>> {
        let mut response = self
            .db
            .query(
                "SELECT * FROM fact WHERE deleted = false \
                 ORDER BY created_at DESC LIMIT $limit",
            )
            .bind(("limit", limit as i64))
            .await?;
        let facts: Vec<Fact> = response.take(0)?;
        Ok(facts)
    }

```

In `crates/alexandria/src/bench.rs` `run()`, change:

```rust
        .list(None, None, false, CORPUS_CAP, 0)
```

to:

```rust
        .live_facts(CORPUS_CAP)
```

and change the `CORPUS_CAP` doc comment:

```rust
/// Most facts read from the corpus in one `list` call. `MemoryRepo::list` orders newest-first,
/// so a corpus at or past this size would silently drop the *oldest* facts — the baseline
/// window and every frozen `QUESTIONS` target — and `run()` bails instead.
```

to:

```rust
/// Most facts read from the corpus in one `MemoryRepo::live_facts` call, which orders
/// newest-first, so a corpus at or past this size would silently drop the *oldest* facts — the
/// baseline window and every frozen `QUESTIONS` target — and `run()` bails instead.
```

`live_facts` gets no test of its own (spec, Testing). It is the query `list` ran for the bench before this change, minus `START 0`, and Step 10's workspace clippy compiles its only caller. The `bench.rs` unit tests are pure math and never call `run()`.

- [ ] **Step 10: Run the gate and the storage tests**

Run: `just fmt-fix && just fmt && cargo clippy --workspace --all-targets --all-features -- -D warnings && cargo test -p alexandria-storage --all-features test_list_`
Expected: fmt and clippy clean, including `crates/alexandria/src/bench.rs`; `test_list_and_count_facts` and `test_list_sorts_by_heat_and_keeps_facts_without_heat` PASS.

Run: `cargo test -p alexandria-mcp --all-features memories`
Expected: all existing memories handler tests PASS (output unchanged).

- [ ] **Step 11: Confirm the change**

Run: `jj st`
Expected: the six files above modified in the `feat(storage): …` change, nothing else.

---

### Task 2: Debug page — heat columns, sortable headers, sort-preserving links

**Files:**
- Modify: `crates/alexandria-mcp/src/debug/memories.rs:1-188` (imports, `COLUMNS`, `sort_params`, `memories_url`, `list`; line numbers after Task 1's import) and its `mod tests`
- Modify: `README.md:117-118` (Memories bullet)
- Modify: `AGENTS.md:72` (test count)

**Interfaces:**
- Consumes (from Task 1): `alexandria_storage::repos::{FactSort, SortColumn}`, `MemoryRepo::list(..., sort: FactSort, limit, offset) -> Result<Vec<FactListRow>>`, `FactListRow.access_count: Option<i64>`, `FactListRow.last_touched: Option<DateTime<Utc>>`, `HeatRepo::create_for_memory`, `HeatRepo::update`.
- Produces: URL contract `/debug/memories?sort=<id|content|tags|confidence|created|access_count|last_touched>&dir=<asc|desc>`.

- [ ] **Step 1: Start the jj change**

```bash
jj new -m "feat(debug): access count and last touched columns, sortable memories table

Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 2: Write the failing handler tests**

In `crates/alexandria-mcp/src/debug/memories.rs` `mod tests`, add after the `use tower::ServiceExt;` line:

```rust

    /// GET `uri` against a fresh router and return the 200 body as text.
    async fn body_of(server: crate::AlexandriaServer, uri: &str) -> String {
        let response = crate::debug::router(server)
            .oneshot(Request::builder().uri(uri).body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(response.status(), 200);
        let body = axum::body::to_bytes(response.into_body(), usize::MAX)
            .await
            .unwrap();
        String::from_utf8(body.to_vec()).unwrap()
    }
```

Then add these tests at the end of `mod tests` (before its closing `}`):

```rust
    #[tokio::test]
    async fn test_memories_list_shows_access_count_and_last_touched() {
        let server = super::super::test_support::test_server().await;
        let repo = alexandria_storage::repos::MemoryRepo::new(server.db.inner());
        let heat_repo = alexandria_storage::repos::HeatRepo::new(server.db.inner());
        let id = repo
            .create_fact("accessed memory", 0.5, &[0.1, 0.2], &[])
            .await
            .unwrap();
        let heat_id = heat_repo.create_for_memory(&id, 1.0).await.unwrap();
        heat_repo.update(&heat_id, 1.0, 1.0, 7).await.unwrap();
        repo.create_fact("never accessed", 0.5, &[0.3, 0.4], &[])
            .await
            .unwrap();

        let text = body_of(server, "/debug/memories").await;
        assert!(text.contains("Access Count"), "expected Access Count header");
        assert!(text.contains("Last Touched"), "expected Last Touched header");
        assert!(text.contains("<td>7</td>"), "expected the access count cell");
        assert!(
            text.contains("<td>—</td><td>—</td></tr>"),
            "a fact without heat_state renders dashes in both heat cells"
        );
    }

    #[tokio::test]
    async fn test_memories_list_sorts_by_access_count_both_ways() {
        let server = super::super::test_support::test_server().await;
        let repo = alexandria_storage::repos::MemoryRepo::new(server.db.inner());
        let heat_repo = alexandria_storage::repos::HeatRepo::new(server.db.inner());
        for (content, accesses) in [("hot memory", 9), ("cold memory", 2)] {
            let id = repo
                .create_fact(content, 0.5, &[0.1, 0.2], &[])
                .await
                .unwrap();
            let heat_id = heat_repo.create_for_memory(&id, 1.0).await.unwrap();
            heat_repo.update(&heat_id, 1.0, 1.0, accesses).await.unwrap();
        }

        let desc = body_of(server.clone(), "/debug/memories?sort=access_count&dir=desc").await;
        assert!(desc.find("hot memory").unwrap() < desc.find("cold memory").unwrap());
        assert!(desc.contains("Access Count ▼"));
        assert!(
            desc.contains("sort=access_count&amp;dir=asc"),
            "the active header links to the opposite direction"
        );

        let asc = body_of(server, "/debug/memories?sort=access_count&dir=asc").await;
        assert!(asc.find("cold memory").unwrap() < asc.find("hot memory").unwrap());
        assert!(asc.contains("Access Count ▲"));
    }

    #[tokio::test]
    async fn test_memories_list_unknown_sort_falls_back_to_created() {
        let server = super::super::test_support::test_server().await;
        let repo = alexandria_storage::repos::MemoryRepo::new(server.db.inner());
        repo.create_fact("fallback memory", 0.5, &[0.1, 0.2], &[])
            .await
            .unwrap();

        let text = body_of(server, "/debug/memories?sort=content%3BDROP&dir=sideways").await;
        assert!(text.contains("fallback memory"));
        assert!(text.contains("Created ▼"), "unknown sort falls back to Created, newest first");
    }

    #[tokio::test]
    async fn test_memories_pagination_links_keep_sort() {
        let server = super::super::test_support::test_server().await;
        let repo = alexandria_storage::repos::MemoryRepo::new(server.db.inner());
        repo.create_fact("page one", 0.5, &[0.1, 0.2], &[])
            .await
            .unwrap();
        repo.create_fact("page two", 0.5, &[0.3, 0.4], &[])
            .await
            .unwrap();

        let text = body_of(server, "/debug/memories?sort=access_count&dir=asc&limit=1").await;
        assert!(
            text.contains("offset=1&amp;sort=access_count&amp;dir=asc"),
            "Next link should carry sort and dir"
        );
    }
```

- [ ] **Step 3: Confirm they fail**

Run: `just fmt-fix && just fmt && cargo clippy --workspace --all-targets --all-features -- -D warnings && cargo test -p alexandria-mcp --all-features memories`
Expected: gate clean (the tests compile against Task 1's API). The four new tests FAIL because the handler ignores `sort`/`dir` and renders neither heat cells nor header links yet: `expected Access Count header`; the sort test fails on either its ordering assertion or `Access Count ▼`; `unknown sort falls back to Created, newest first`; `Next link should carry sort and dir`. Existing tests still PASS.

- [ ] **Step 4: Replace the imports, add `COLUMNS` and `sort_params`, update `memories_url`**

Replace everything in `crates/alexandria-mcp/src/debug/memories.rs` from line 1 through the end of `fn memories_url` (the `}` before `pub async fn list(`) with:

```rust
use std::collections::HashMap;

use alexandria_storage::repos::{FactSort, SortColumn};
use axum::extract::{Path, Query, State};
use axum::http::StatusCode;
use axum::response::{Html, IntoResponse};

use super::html::{esc, layout};
use crate::AlexandriaServer;
use crate::server::record_id_to_string;

/// Memories table columns in display order: URL `sort` key, header label, storage column.
/// The only path from a request's `sort` param to the query.
const COLUMNS: [(&str, &str, SortColumn); 7] = [
    ("id", "ID", SortColumn::Id),
    ("content", "Content", SortColumn::Content),
    ("tags", "Tags", SortColumn::Tags),
    ("confidence", "Confidence", SortColumn::Confidence),
    ("created", "Created", SortColumn::Created),
    ("access_count", "Access Count", SortColumn::AccessCount),
    ("last_touched", "Last Touched", SortColumn::LastTouched),
];

/// URL `sort` and `dir` values for a sort order.
fn sort_params(sort: FactSort) -> (&'static str, &'static str) {
    let key = COLUMNS
        .iter()
        .find(|c| c.2 == sort.column)
        .map(|c| c.0)
        .expect("every SortColumn has a COLUMNS entry");
    (key, if sort.descending { "desc" } else { "asc" })
}

/// Build a URL back to the memories list with the given params.
/// Simple encoding — debug UI only, not production.
fn memories_url(
    search: Option<&str>,
    tag: Option<&str>,
    include_deleted: bool,
    sort: FactSort,
    limit: usize,
    offset: usize,
) -> String {
    let mut parts = vec![format!("limit={limit}"), format!("offset={offset}")];
    if let Some(s) = search {
        parts.push(format!(
            "search={}",
            s.replace('&', "%26").replace(' ', "+")
        ));
    }
    if let Some(t) = tag {
        parts.push(format!("tag={}", t.replace('&', "%26").replace(' ', "+")));
    }
    if include_deleted {
        parts.push("include_deleted=true".to_string());
    }
    let (sort_key, dir) = sort_params(sort);
    parts.push(format!("sort={sort_key}&dir={dir}"));
    format!("/debug/memories?{}", parts.join("&"))
}
```

- [ ] **Step 5: Parse the sort in `list` and pass it to the repo**

In `pub async fn list`, directly after the `let offset: usize = ...;` statement, insert:

```rust
    let default_sort = FactSort::default();
    let sort = FactSort {
        column: params
            .get("sort")
            .and_then(|key| COLUMNS.iter().find(|c| c.0 == key.as_str()))
            .map_or(default_sort.column, |c| c.2),
        descending: params
            .get("dir")
            .map_or(default_sort.descending, |dir| dir != "asc"),
    };
```

and in the `repo.list(` call change the `FactSort::default(),` argument (added in Task 1) to `sort,`.

- [ ] **Step 6: Render the two new cells**

In the `for fact in &rows` loop, after the `let created = ...;` statement, insert:

```rust
        let access_count = fact
            .access_count
            .map(|n| n.to_string())
            .unwrap_or_else(|| "—".to_string());
        let last_touched = fact
            .last_touched
            .map(|dt| dt.format("%Y-%m-%d %H:%M UTC").to_string())
            .unwrap_or_else(|| "—".to_string());
```

and replace the row `format!` string:

```rust
            r#"<tr{row_class}><td><a class="link" href="/debug/memories/{id_esc}">{id_esc}</a></td><td>{}</td><td>{}</td><td>{:.2}</td><td>{created}</td></tr>"#,
```

with:

```rust
            r#"<tr{row_class}><td><a class="link" href="/debug/memories/{id_esc}">{id_esc}</a></td><td>{}</td><td>{}</td><td>{:.2}</td><td>{created}</td><td>{access_count}</td><td>{last_touched}</td></tr>"#,
```

- [ ] **Step 7: Build the sortable header and wire sort into the body**

Directly before `let body = format!(`, insert:

```rust
    let header: String = COLUMNS
        .iter()
        .map(|&(_, label, column)| {
            let active = column == sort.column;
            // Inactive columns open descending; the active one flips.
            let next = FactSort {
                column,
                descending: !(active && sort.descending),
            };
            let arrow = match (active, sort.descending) {
                (false, _) => "",
                (true, true) => " ▼",
                (true, false) => " ▲",
            };
            let href = memories_url(
                search.map(|s| s.as_str()),
                tag.map(|s| s.as_str()),
                include_deleted,
                next,
                limit,
                0,
            );
            format!(
                r#"<th><a class="link" href="{}">{label}{arrow}</a></th>"#,
                esc(&href)
            )
        })
        .collect();
    let (sort_key, dir) = sort_params(sort);
```

In the `body` format string, replace:

```html
<label><input type="checkbox" name="include_deleted" value="true" {}> include deleted</label>
</form>
```

with:

```html
<label><input type="checkbox" name="include_deleted" value="true" {}> include deleted</label>
<input type="hidden" name="sort" value="{sort_key}">
<input type="hidden" name="dir" value="{dir}">
</form>
```

and replace:

```html
<tr><th>ID</th><th>Content</th><th>Tags</th><th>Confidence</th><th>Created</th></tr>
```

with:

```html
<tr>{header}</tr>
```

In both `prev_link` and `next_link` `memories_url(...)` calls, add `sort,` after `include_deleted,`:

```rust
                esc(&memories_url(
                    search.map(|s| s.as_str()),
                    tag.map(|s| s.as_str()),
                    include_deleted,
                    sort,
                    limit,
                    prev_offset
                ))
```
```rust
                esc(&memories_url(
                    search.map(|s| s.as_str()),
                    tag.map(|s| s.as_str()),
                    include_deleted,
                    sort,
                    limit,
                    offset + limit
                ))
```

- [ ] **Step 8: Run the gate and handler tests**

Run: `just fmt-fix && just fmt && cargo clippy --workspace --all-targets --all-features -- -D warnings && cargo test -p alexandria-mcp --all-features memories`
Expected: gate clean; all memories tests PASS, including the four new ones and the existing `test_memories_list_shows_created_at` (it still finds `Created` and `UTC`) and `test_memories_list_shows_total_count_and_pagination` (no `Prev` on page one).

- [ ] **Step 9: Update the README**

In `README.md`, replace:

```markdown
- **Memories** (`/debug/memories`) — paginated search/filter of facts by content and tag; click through to a
  detail view showing heat, stability, timestamps, cluster membership, and graph edges
```

with:

```markdown
- **Memories** (`/debug/memories`) — paginated search/filter of facts by content and tag, with access count
  and last-touched columns; click any column header to sort the whole result set. Click through to a
  detail view showing heat, stability, timestamps, cluster membership, and graph edges
```

- [ ] **Step 10: Full suite**

Run: `just fmt && cargo clippy --workspace --all-targets --all-features -- -D warnings && just test`
Expected: everything green on stable, including Task 1's new storage test and this task's four handler tests. No other test changes.

- [ ] **Step 11: Update the AGENTS.md test count**

Feature commits keep the count in AGENTS.md's Testing section current. There are no `#[ignore]` tests, so the listed count equals the passed count:

Run: `cargo test --workspace --all-features -- --list 2>/dev/null | grep -c ': test$'`

In `AGENTS.md`, change `Current suite: 166 tests, all green.` to that number. 166 is the value written before this work, so use the command's output rather than 166 + 5.

- [ ] **Step 12: Confirm the change**

Run: `jj st`
Expected: `crates/alexandria-mcp/src/debug/memories.rs`, `README.md` and `AGENTS.md` modified in the `feat(debug): …` change, nothing else.

---

## After merge (manual, user)

The spike timings are in-memory. After the live server is rebuilt and restarted, time the heaviest
page against the real `surrealkv` store:

```bash
curl -s -o /dev/null -w '%{time_total}s\n' 'http://127.0.0.1:3000/debug/memories?sort=access_count&dir=desc&include_deleted=true'
```

If this is uncomfortably slow, the `ponytail:` comment in `MemoryRepo::list` names the upgrade
path.
