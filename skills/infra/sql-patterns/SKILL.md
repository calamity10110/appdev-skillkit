---
name: sql-patterns
description: Designing and reviewing relational database work — schema design, indexing strategy, transaction discipline, and migration safety — for PostgreSQL and other relational platforms.
---

# SQL Patterns

Rules of thumb for building and reviewing relational database schemas and
queries, drawn from production practice on PostgreSQL and SQL Server/Azure
SQL. The defaults below apply to both; platform specifics are marked.

## Schema design

- Every table gets a primary key. Prefer narrow, stable, increasing
  surrogates (bigint identity in PG; int/bigint identity in SQL Server).
  If a UUID key is needed, use a time-ordered variant (uuidv7 in PG 18+)
  rather than random UUIDs, which fragment indexes.
- Use correct native types from the start: `timestamptz` (never bare
  timestamp), exact decimal types for money (never float), `text` rather
  than arbitrary `varchar(n)` limits in PG. Never store numbers, dates, or
  booleans as strings; never pack multiple values into one column.
- Add `UNIQUE` constraints for every natural key the business relies on,
  `CHECK` constraints for domain rules, and `NOT NULL` everywhere a value
  is logically required. Constraints are cheaper than the bugs they
  prevent; "validate in the app instead" is a false economy when there is
  more than one writer.
- Foreign keys on by default, with the `ON DELETE` behaviour chosen
  deliberately. On PostgreSQL, index every FK column — PG does not
  auto-index them, and joins plus parent deletes pay for it.
- Name constraints and indexes deterministically
  (`pk_users`, `uq_users_email`, `users_email_idx`): auto-generated names
  make migrations and drift comparison painful.
- Default to 3NF for transactional schemas; denormalise only with a
  measured read-path justification, documented along with the update
  anomaly you accepted. Wide "god tables" with 100+ nullable columns are
  a smell. Reporting workloads belong in a read model or star schema, not
  in ever-wider joins over the OLTP schema.

## Indexing strategy

- Design indexes per workload: equality predicates lead the key (most
  selective first), then the single range predicate, then `ORDER BY`/`GROUP
  BY` columns to avoid sorts; output-only columns go in the covering
  clause (`INCLUDE`) rather than the key.
- Fix predicate SARGability first — an index cannot help a column wrapped
  in a function or fighting a type/collation mismatch.
- Never recommend an index without evidence: the actual execution plan,
  row counts, and the existing index DDL. Merge overlapping candidates
  and delete duplicates before adding; count the write cost (every index
  taxes every insert/update/delete).
- Prefer composite covering indexes over stacks of single-column indexes;
  be wary of filtered/partial indexes under parameterised queries, where
  the planner may be unable to prove the filter applies.
- Maintenance is a system, not a one-off: do not disable autovacuum (PG),
  watch statistics freshness after bulk loads (stale stats break plans far
  more often than fragmentation does), and use `CREATE INDEX CONCURRENTLY`
  (PG) or online index operations (SQL Server) against hot tables.

## Transactions and concurrency

- Keep transactions short. Never hold one open across user interaction,
  HTTP calls, or queue waits.
- Acquire locks in a consistent order across all code paths touching the
  same rows — that alone eliminates most deadlocks.
- Deadlock victims and serialisation failures are transient: retry the
  whole transaction at the application level with bounded exponential
  backoff, never the failed statement in isolation.
- Work-queue pickup without convoying: `SELECT ... FOR UPDATE SKIP
  LOCKED` (PG) so competing workers skip rows already claimed.
- Detect lost updates without long locks via optimistic concurrency —
  a row-version column compared on write — and surface a conflict to the
  user rather than silently last-writer-wins.
- Never "fix" blocking with dirty reads (`NOLOCK`/`READ UNCOMMITTED`):
  they return dirty, duplicated, and missing rows. Fix the blocking
  cause instead.

## Migration discipline

- Forward-only, idempotent, versioned, reviewed as code, applied by the
  pipeline — never run by hand against production, and never
  auto-migrated on application startup.
- Anything a live application reads must change as expand → migrate →
  contract: additive change first (both old and new code work), backfill
  and flip, then remove the old path once provably unused. Renames are
  drop+add in disguise; bridge them with a compatibility view.
- Keep locking-aware DDL: concurrent index builds, `NOT VALID` foreign
  keys validated later, and `lock_timeout` set in the migration session
  so a stuck DDL retries instead of wedging production.
- Backfills are migrations too: batched, resumable, throttled, and
  verified by row-count or checksum comparison before cutover. Keep data
  migrations separate from schema migrations so each can be batched,
  retried, and reviewed independently.
- The same artefact flows dev → test → prod; per-environment differences
  are parameters, not edited scripts. And a backup only exists when a
  restore has been proven — test restores on a schedule.

## Query habits

- Parameterise everything. No string-concatenated SQL in application
  code, dynamic SQL, or examples — injection-unsafe code is a defect even
  in a draft.
- Set-based, not row-by-row: cursors and per-row loops need written
  justification; rewrite as joins, window functions, or batched DML.
- Pagination by keyset (cursor) on an indexed key, not deep `OFFSET`.
- Existence tests with `EXISTS`, not `COUNT(*) > 0`.
- `ON CONFLICT DO UPDATE` (PG) / guarded upsert for write-once-then-update
  patterns; predicates on the optional side of a `LEFT JOIN` belong in the
  `ON` clause, not `WHERE`, or the outer join silently becomes an inner
  join.

## Input → Process → Output

**INPUT:** "Split the `full_name` column on `users` into `first_name` /
`last_name`, and add a `status` enum; 40M rows, zero-downtime deploy."

**PROCESS:**
1. Design: `first_name`/`last_name` nullable text, `status` as a native
   enum with a CHECK-compatible default; naming stays snake_case, PK stays
   `id bigint generated always as identity`.
2. Expand: migration 1 adds the new columns (nullable, no default, no
   backfill) — safe with old code running.
3. Migrate: backfill in batches (10k rows, `WHERE id > $1`, short
   transactions) with the app writing both old and new columns.
4. Contract: migration 2 drops `full_name` after two deploys; add
   `NOT NULL` to the new columns once the backfill is verified.
5. Indexing: workload-aware — index `(status, created_at)` for the admin
   queue query; verify with `EXPLAIN (ANALYZE, BUFFERS)` on production-like
   data, not assumptions.

**OUTPUT:** A two-migration expand→migrate→contract plan with batched
backfill, rollback point (revert migration 2 only), and an evidence-backed
index list.

## Proven pattern vs avoid pattern

- **Proven — expand→migrate→contract:** additive changes first, data
  migration while both shapes coexist, destructive drops last. Old and new
  code both run against every intermediate schema, so deploys are boring.
- **Avoid — the big-bang ALTER with a default on a huge table:** adding
  `NOT NULL ... DEFAULT` rewrites all 40M rows in one transaction, takes an
  exclusive lock, and blocks the deploy. Backfill in batches, add the
  constraint after.

## Telemetry, testing & debugging

- **Instrument:** migration duration per batch, lock-wait times, table bloat
  after the rewrite; slow-query log with `auto_explain` for regressions.
- **Test:** dry-run every migration against a production-sized snapshot in
  a transaction (`BEGIN; ... ROLLBACK;`) or a cloned database; run the app
  test suite against both the pre- and post-migration schema.
- **Debug:** `EXPLAIN (ANALYZE, BUFFERS)` on the actual slow query to see
  whether the planner used your index; `pg_stat_activity` + `pg_locks` to
  find what's holding the lock during a stuck migration.
