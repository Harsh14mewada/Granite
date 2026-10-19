# 🪨 GRANITE — Learn Database Internals by Building

### Python Edition — Build a small SQL database from scratch, then grow it deliberately

*One learning project, 13 phases (0–12). Finish a working database early; add deeper internals and client compatibility one layer at a time.*

**Purpose:** understand and implement the database myself. Notes in my copy first, then code written by me. This roadmap guides the work; implementation remains my exercise.

**Legend:**

- 🐍 = Implement the database logic myself in Python
- 🔧 = Use an existing tool for development, testing, or operations
- 🧪 = Break it deliberately and explain the result
- 📖 = Study the concept, reason about the design, and document the decision

**Learning loop:** read the concept/spec → predict the behavior → write notes and a small design → implement independently → test/break it → compare with a reference implementation → explain the tradeoffs.

**Rules:**

- One small, observable milestone at a time; finish its exit check before expanding scope.
- Read concepts and protocol specifications before implementing. Compare implementation code after my own attempt; no blind copying.
- Use Python's standard library for bytes, files, sockets, and synchronization. Build the storage engine, parser, executor, indexes, and transaction logic myself; do not delegate them to SQLite or another database.
- Test correctness in every phase. Advanced fuzzing comes later, but basic invariants start immediately.
- Git-tag each completed phase and write one short explanation: what I built, how it works, how I tested it, and what I left out and why.
- Record guarantees precisely. Clean restart, process crash, machine crash, and power loss are different failure models.
- Keep a supported-feature list. Unsupported SQL/features should fail clearly.

**First finish line — end of Phase 2:** my local CLI runs `CREATE TABLE`, `INSERT`, and `SELECT ... WHERE`. After a clean close and reopen, the rows are still there. I can locate a row's bytes on disk and explain how the query found it.

**Scope for that first version:** one process, one connection, no concurrent transactions, `INTEGER` and `TEXT`, heap scans, a small persistent catalog, and a simple REPL. Crash recovery, indexes, networking, and ORM compatibility come afterward.

**Long-term direction:** own CLI → crash-recoverable engine → indexed queries → concurrent transactions → `psql` → psycopg → a defined Django/FastAPI example → optional self-hosting. Each arrow is a separate milestone; PostgreSQL wire compatibility does not imply PostgreSQL SQL/catalog compatibility.

**References — concepts/specifications before building, implementation comparisons after an independent attempt:**

- 🎥 [CMU 15-445 Intro to Database Systems](https://www.youtube.com/playlist?list=PLSE8ODhjZXjbohkNBWQs_otTrBTrjyohi) — storage, indexes, execution
- 🎥 [CMU 15-721 Advanced Database Systems](https://www.youtube.com/playlist?list=PLSE8ODhjZXjasmrEd2_Yi1deeE360zv5O) — concurrency and recovery
- 📖 [Database Internals](https://www.databass.dev) — storage engines and tree structures
- 📖 [Designing Data-Intensive Applications](https://dataintensive.net/) — system tradeoffs
- 🧬 [toyDB](https://github.com/erikgrinaker/toydb), its [architecture guide](https://github.com/erikgrinaker/toydb/blob/main/docs/architecture/index.md), and [references](https://github.com/erikgrinaker/toydb/blob/main/docs/references.md) — implementation comparison
- 📄 [PostgreSQL 16 protocol specification](https://www.postgresql.org/docs/16/protocol.html) — fixed reference target for Phase 7; pin and record client versions separately
- 🐛 [SQLancer](https://github.com/sqlancer/sqlancer) — advanced logic-bug testing in Phase 9

---

## PHASE 0 — Bootstrap + Bytes on Disk

- [ ] Repo `granite`, README, pyproject (uv), ruff, pytest; start with one main branch
- [ ] 📖 ADR-001: 8KB pages, explicit on-disk little-endian encoding, UTF-8 text, one heap file per table; explain why these are choices rather than universal rules
- [ ] 📖 ADR-002: typed internal values; start with non-null `INTEGER` and `TEXT`. Wire encoding will be a separate concern
- [ ] 📖 ADR-003: one process and one connection initially; defer server/threading decisions
- [ ] 🐍 Byte-format exercise: encode/decode an integer and a length-prefixed UTF-8 string without pickle
- [ ] 🧪 Compare Python buffered writes, `flush`, `os.write`, and `fsync`; distinguish user-space buffers from OS caches and durable storage
- [ ] 🧪 Handle partial reads/writes and truncated input explicitly
- [ ] Minimal layout: `granite/storage/`, `tests/`, `docs/adr/`, `docs/writeups/`; add other modules when needed
- [ ] 📖 Writeup: "What happens between my Python value and a byte on disk?"
- [ ] **Exit check:** round-trip known values, inspect the bytes manually, and reject malformed/truncated input
- [ ] Tag `v0.1.0-bootstrap`

## PHASE 1 — Persistent Rows: Pages + Heap Files

- [ ] 🐍 Page manager: fixed-size pages, page 0 header, format version, page allocation, explicit bounds checks
- [ ] 🐍 Slotted page: header + slot directory + tuple bytes growing from the end
- [ ] 🐍 Tuple format for a known schema: integer bytes and variable-length text; RowID = `(page_no, slot_no)`
- [ ] 🐍 Heap operations: insert and sequential scan; allocate another heap page when the current page is full
- [ ] 🐍 Free-space tracking: begin with a linear search; a separate free-space map is a later optimization
- [ ] 🐍 Tiny buffer pool after direct page I/O works: pins, dirty flags, LRU eviction, flush/unpin
- [ ] 🧪 Insert across page boundaries, close/reopen, and compare every row with an in-memory reference
- [ ] 🧪 Assert slot bounds, non-overlapping tuple regions, free-space accounting, and pinned-page eviction rules
- [ ] 🧪 Add a page checksum; deliberately corrupt a copied data file and report the bad page clearly. Do not silently skip damaged data
- [ ] 📖 Writeup: "Why pages and slots? What happens when a heap page fills?"
- [ ] **Exit check:** rows survive a clean restart; I can identify their page, slot, and byte range. No crash-safety guarantee yet
- [ ] Tag `v0.2.0-storage`

## PHASE 2 — My First SQL Database

- [ ] 🐍 Minimal persistent catalog: table names, column names/types, and heap-file mapping; a simple bootstrap metadata format is enough
- [ ] 🐍 Raw engine API: create table, insert row, scan table; SQL calls this API
- [ ] 🐍 Lexer + hand-written parser → AST for `CREATE TABLE`, `INSERT`, and `SELECT ... FROM ... WHERE`
- [ ] 🐍 Binder: resolve table/column names and validate values against the schema
- [ ] 🐍 Executor: heap scan → filter → projection; start with equality predicates and explicit column lists or `*`
- [ ] 🐍 Local REPL: execute a statement, print rows/errors, close cleanly
- [ ] 🧪 Golden SQL tests: expected rows and expected errors, including unknown columns, invalid types, empty results, and quoted text
- [ ] 🧪 End-to-end script: create → insert → query → close → reopen → query again
- [ ] 📖 Writeup: trace one SQL statement through tokens, AST, binding, page reads, and result rows
- [ ] **Exit check — first complete database:** `CREATE TABLE`, `INSERT`, and `SELECT ... WHERE` work through my CLI and survive clean restart
- [ ] Tag `v0.3.0-tiny-sql`

## PHASE 3 — Single-Transaction WAL + Crash Recovery

- [ ] 📖 Define transaction states, commit acknowledgement, rollback behavior, and the supported failure model before writing recovery code
- [ ] 🐍 `BEGIN`, `COMMIT`, `ROLLBACK`, and autocommit with only one active transaction
- [ ] 📖 Choose a simple initial recovery design in an ADR. Suggested starting point: no uncommitted page reaches the heap (no-steal), with committed page after-images in a redo log
- [ ] 🐍 Stage changes privately; rollback discards them. Bound transaction size so dirty uncommitted pages never need eviction to disk
- [ ] 🐍 WAL framing: length, record type, transaction ID, LSN, checksum, page identity, page after-image, and commit record
- [ ] 🐍 Commit order: append all required records → append commit record → fsync WAL → acknowledge commit; data-page writes follow durable WAL
- [ ] 🐍 Recovery: replay complete committed transactions, ignore incomplete transactions, and safely remove an incomplete final log record before appending. Report unexpected interior corruption
- [ ] 🐍 Make replay repeatable, including page allocation/header changes. Crash again during recovery and reopen successfully
- [ ] 🐍 First checkpoint: pause transaction admission, finish active work, flush/fsync committed state, then safely retire old WAL. Defer fuzzy checkpoints and group commit
- [ ] 🐍 Start with DML on existing tables. Extend the design to crash-safe catalog/file creation before claiming transactional DDL; define file creation/deletion recovery explicitly
- [ ] 🧪 Subprocess crash tests at controlled points: before WAL sync, after WAL sync, during data-page flush, during checkpoint, and during recovery
- [ ] 🧪 Acknowledged commits must survive tested process crashes; incomplete transactions must disappear. A durable commit may exist even if its acknowledgement never reached the client
- [ ] 📖 Writeup: explain WAL ordering and why `kill -9` tests do not simulate loss of the OS page cache or prove power-loss safety
- [ ] **Exit check:** committed DML recovers and incomplete DML disappears under the documented model; rollback works; transactional DDL has its own explicit completion check
- [ ] Tag `v0.4.0-recovery`

## PHASE 4 — B+Tree Indexes

- [ ] 🐍 B+tree page format, search, leaf insert, node/root splits, and linked-leaf range scans
- [ ] 🧪 Build and test the tree independently before wiring it into SQL
- [ ] 🐍 Index catalog entries; start with one integer-key index and define duplicate-key behavior
- [ ] 🐍 Maintain heap and index changes in the same transaction; include index pages and root metadata in WAL/recovery
- [ ] 🐍 Unique and non-unique indexes; define enforcement semantics explicitly
- [ ] 🐍 Index scan executor; simple planner choice between heap scan and index scan; basic `EXPLAIN`
- [ ] 🧪 Check sorted keys, occupancy rules, child ranges, leaf links, duplicate keys, and root splits
- [ ] 🧪 Compare index results against heap scans for randomized inserts/ranges, including after crashes
- [ ] 🧪 Start benchmarks at 1k/10k rows, then increase; measure cold/warm conditions and validate results before comparing timings
- [ ] 📖 Defer deletion/rebalancing until insert/search/recovery are correct. Implement index deletion before enabling SQL delete/update of indexed rows
- [ ] 📖 Writeup: "Heap-page allocation versus B+tree splitting; why an index can avoid a full scan"
- [ ] **Exit check:** indexed queries return the same rows as heap scans and remain correct after recovery
- [ ] Tag `v0.5.0-indexes`

## PHASE 5 — Concurrent Transactions

- [ ] 🐍 Introduce multiple engine sessions with independent transaction state; networking is still optional
- [ ] 🐍 Begin with a coarse global transaction lock: serialize transactions and establish a correctness baseline
- [ ] 📖 Revisit Phase 3's single-transaction assumptions: transaction IDs, buffer ownership, WAL ordering, checkpoint coordination, and index consistency
- [ ] 📖 Choose and document the next locking design: row/table locks, lock lifetimes, lock ordering, and an isolation level
- [ ] 🐍 Add finer-grained locks incrementally; protect buffer-pool, catalog, and B+tree internal structures separately from transaction-level locks
- [ ] 🐍 If blocking on multiple locks is possible, implement one deadlock strategy: detection or prevention
- [ ] 🧪 Deterministic session schedules: dirty reads, rollback visibility, concurrent increments, conflicts, and deadlocks where applicable
- [ ] 📖 Record exactly which anomalies the chosen isolation level allows. READ COMMITTED alone does not promise to prevent every lost-update pattern
- [ ] 🧪 Repeat recovery tests with interleaved transaction log records
- [ ] 📖 Writeup: "Why the GIL is not a transaction manager; what my isolation level guarantees"
- [ ] **Exit check:** concurrent behavior matches the documented isolation/locking rules, with tested recovery
- [ ] Tag `v0.6.0-concurrency`

## PHASE 6 — A More Useful SQL Engine

- [ ] 🐍 Add `BOOLEAN`, `BIGINT`, and nullable columns; implement SQL NULL and three-valued predicate logic explicitly
- [ ] 🐍 Expression precedence, comparisons, AND/OR/NOT, and `IS NULL`
- [ ] 🐍 `UPDATE`, `DELETE`, and `DROP TABLE`, with catalog/index/WAL integration; complete B+tree deletion maintenance as needed
- [ ] 🐍 Iterator operators: scan, filter, projection, sort, limit; add `ORDER BY` and `LIMIT`
- [ ] 🐍 Parameter AST nodes and a separate binding API: values stay values rather than being interpolated into SQL text
- [ ] 🧪 Bind hostile-looking strings and confirm they remain literal data. Placeholders alone do not make application string construction safe
- [ ] 🐍 Add `COUNT`, then other aggregates/grouping and nested-loop joins as separate optional increments
- [ ] 🧪 Golden tests for expression precedence, NULL behavior, empty inputs, statement failures, rollback, and index maintenance
- [ ] 📖 Document statement-error behavior inside a transaction and whether the whole transaction must be rolled back
- [ ] 📖 Writeup: "Parser → binder → plan → executor, and which SQL semantics I actually support"
- [ ] **Exit check:** the published SQL subset works through the CLI and passes correctness/recovery tests
- [ ] Tag `v0.7.0-sql`

## PHASE 7 — PostgreSQL Wire Protocol: psql, Then psycopg

- [ ] 📖 Pin protocol v3.0 and record tested `psql`/psycopg versions; keep a protocol/SQL compatibility matrix
- [ ] 🐍 Localhost server, initially one client at a time; length-delimited message framing, partial socket reads, and message-size limits
- [ ] 🐍 StartupMessage, local trust authentication, required status messages, and ReadyForQuery
- [ ] 🐍 Handle startup negotiation requests from the chosen client explicitly, including declining SSL when TLS is unavailable
- [ ] 🐍 Simple query path: Query → RowDescription → DataRow → CommandComplete → ReadyForQuery
- [ ] 🐍 Network byte order for protocol fields; keep it separate from the heap format's little-endian encoding
- [ ] 🐍 Map supported internal types to PostgreSQL OIDs and text encodings. PostgreSQL supports both text and binary formats; only claim formats actually implemented
- [ ] 🧪 **Milestone A:** `psql -h localhost -p 5433` connects and executes supported SQL; do not assume psql meta-commands/catalog queries work automatically
- [ ] 🐍 Extended path for the pinned client: Parse, Bind, Describe, Execute, Sync, statement/portal state, and required cleanup/error responses
- [ ] 🐍 Implement requested parameter/result formats required by that client; reject unsupported combinations clearly
- [ ] 🐍 ErrorResponse with accurate SQLSTATEs: `42P01` undefined table, `23505` unique violation, `25P01` no active transaction, `25P02` failed transaction
- [ ] 🐍 ReadyForQuery `I`/`T`/`E`, rollback on disconnect, and recovery after protocol/statement errors
- [ ] 🧪 **Milestone B:** a small psycopg script connects, binds parameters, performs CRUD, commits, and rolls back
- [ ] 📖 Writeup: trace the actual message exchange of the tested client; separate wire support from dialect/catalog support
- [ ] **Exit check:** the exact documented psql and psycopg examples work without custom client code
- [ ] Tag `v0.8.0-protocol`

## PHASE 8 — Defined App Compatibility

- [ ] 📖 Choose and pin one Django version and one driver version; start with one small model and a declared migration set
- [ ] 🧪 Capture connection/setup SQL, migration SQL, and introspection queries from that exact example; turn each required behavior into a test
- [ ] 🐍 Implement the required types, constraints, defaults/generated IDs, schema changes, savepoints, and catalog queries incrementally
- [ ] 🐍 Implement version/settings functions only with documented semantics; avoid advertising unsupported PostgreSQL capabilities
- [ ] 🧪 **Milestone A:** the selected Django project's migrations and model save/query/update/delete run against Granite
- [ ] 🧪 **Milestone B:** a pinned FastAPI + SQLAlchemy/psycopg example performs CRUD; test its dialect/setup SQL independently
- [ ] 📖 Maintain an explicit matrix: works, partially supported, unsupported. One successful project does not imply arbitrary Django/PostgreSQL compatibility
- [ ] 📖 Writeup: "What my ORM asked the database to do beyond sending SQL over a socket"
- [ ] **Exit check:** the defined examples pass repeatable integration tests with unmodified client libraries
- [ ] Tag `v0.9.0-app-compatibility`

## PHASE 9 — Advanced Correctness + Fuzzing

- [ ] 🔧 Property-based tests with Hypothesis: page operations, tree operations, transaction sequences, and the supported SQL subset
- [ ] 🐍 Compare against an independent reference model; use another SQL engine only where dialect/NULL/type semantics match
- [ ] 🐍 Metamorphic checks: index versus heap scan, equivalent predicates, and restart/recovery equivalence
- [ ] 📖 Study NoREC before implementing its reference-query transformation; disabling optimization alone is not the full oracle
- [ ] 🐍 Ternary logic partitioning for supported queries: compare `WHERE p` with the multiset union of `WHERE p AND x`, `WHERE p AND NOT x`, and `WHERE p AND x IS NULL`; account for ordering, duplicates, limits, and aggregates
- [ ] 🔧 Integrate SQLancer only after mapping supported SQL and required provider behavior. PG wire support alone is insufficient for its PostgreSQL provider's full query generator
- [ ] 🧪 Minimize each failure, add a regression, fix it, and rerun the relevant tests
- [ ] 🧪 Run a recorded fuzz campaign; save seeds, versions, workload, duration, and failures
- [ ] 📖 Writeup: "Which bugs my oracles found, and which bugs they could still miss"
- [ ] **Exit check:** a defined test/fuzz campaign completes without known failures. This is evidence, not proof of zero bugs
- [ ] Tag `v0.10.0-fuzzing`

## PHASE 10 — Server Operations + Backup/Restore

- [ ] 🐍 Bounded concurrent connections, statement/resource limits, config file, graceful shutdown, and a `granited` CLI
- [ ] 📖 Choose thread-per-connection or a worker design based on the engine's synchronization model; measure before changing it
- [ ] 🔧 Structured logs and slow-query logging; metrics for connections, transaction outcomes, WAL bytes, buffer hits, and query latency
- [ ] 🐍 First physical backup: stop writes, cleanly shut down and sync state, copy the complete data/catalog/WAL set, then restore into a separate directory
- [ ] 🧪 Backup drill: restore elsewhere, compare schema/rows, and time recovery. Protect the original data during the drill
- [ ] 🐍 Optional logical export/import in Granite's documented format; distinguish it from pg_dump compatibility
- [ ] 📖 Online backups require a separate consistency design: a checkpoint followed by copying changing files is not sufficient
- [ ] 🧪 Driver-based load test with a recorded dataset, concurrency, latency distribution, and correctness checks; publish the measured ceiling
- [ ] 📖 Writeup: "My operating limits and what a successful restore actually proves"
- [ ] **Exit check:** documented startup/shutdown, bounded resource behavior, and a verified backup/restore path
- [ ] Tag `v0.11.0-operations`

## PHASE 11 — Optional Self-Hosted Demo

- [ ] 🔧 Docker/compose or systemd deployment with persistent storage; pin and document the environment
- [ ] 🔧 Configure authenticated, encrypted access before exposing the server beyond its trusted local environment
- [ ] 🔧 CI: lint → core tests → crash/recovery tests → pinned-client/app integration tests → build
- [ ] 🔧 Monitoring and offsite backups; restore into a fresh environment and record the result
- [ ] 🧪 Run a disposable demo app for two weeks. Use Linkly only after its actual schema/query requirements pass the compatibility matrix
- [ ] 🧪 Restart the service, restore a backup, fill a test disk, and record the observed behavior
- [ ] 📖 Writeup: "What broke during deployment, what I measured, and what remains unsupported"
- [ ] **Exit check:** the demo runs with documented limits and a reproducible restore procedure
- [ ] Tag `v0.12.0-hosted-demo`

## PHASE 12 — Stretch: Pick One Question at a Time

- [ ] 🐍 Group commit, WAL segmentation, fuzzy checkpoints, and a more advanced recovery algorithm; revisit buffering and recovery assumptions together
- [ ] 🐍 MVCC, snapshots, version visibility, and garbage collection/VACUUM
- [ ] 🐍 More SQL: joins, aggregates, richer types, constraints, and a cost-based optimizer
- [ ] 🐍 B+tree deletion/rebalancing refinements, free-space maps, and page compaction
- [ ] 🐍 Streaming WAL replication and a documented failover/data-loss model
- [ ] 🐍 Native TLS, SCRAM-SHA-256, and a broader authentication model
- [ ] 🐍 COPY for bulk loading; set performance targets after collecting a baseline
- [ ] 🐍 Broader pg_catalog, pg_dump/pg_restore, and pgAdmin compatibility; evaluate each as a distinct project
- [ ] 📖 Raft/sharding design study, or reimplement a small engine component in Rust and compare
- [ ] 📖 For each chosen extension: state the question, prerequisites, testable guarantee, and tradeoff before coding

---

## 🏁 Definition of Done — Three Separate Finish Lines

**First database — Phase 2:**

- [ ] I wrote the engine and can explain one row's full path from SQL to disk and back
- [ ] `CREATE TABLE`, `INSERT`, and `SELECT ... WHERE` work in my CLI
- [ ] Catalog and rows survive a clean restart; limitations are explicit

**Core database internals — Phase 6:**

- [ ] WAL/recovery, indexes, transactions, and the published SQL subset work together
- [ ] Deterministic crash tests preserve acknowledged commits and discard incomplete transactions under the stated failure model
- [ ] Index/heap results agree; concurrent schedules match the documented isolation rules
- [ ] Every completed phase has tests, a tag, and a writeup ending with "what I left out and why"

**Optional integration/showcase — Phases 7–11:**

- [ ] Pinned psql/psycopg clients and the explicitly selected app examples work without custom client code
- [ ] Fuzz results record coverage, versions, seeds, and duration without claiming proof of correctness
- [ ] A self-hosted demo and a fresh-environment restore are verified
- [ ] README includes architecture, supported SQL/client matrix, measured limits, failure model, and known gaps

## 📏 The Honest Bar

Granite is an educational database with a growing, explicitly tested feature set. Python is a deliberate learning choice. Hosting a demo, passing a fuzz campaign, or using PostgreSQL's protocol does not by itself establish production readiness.

The measure of progress is whether I can explain, implement, test, and debug each layer independently. A small working database I understand is already a completed learning milestone; later phases extend it.
