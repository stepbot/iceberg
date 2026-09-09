<!--
 - Licensed to the Apache Software Foundation (ASF) under one or more
 - contributor license agreements.  See the NOTICE file distributed with
 - this work for additional information regarding copyright ownership.
 - The ASF licenses this file to You under the Apache License, Version 2.0
 - (the "License"); you may not use this file except in compliance with
 - the License.  You may obtain a copy of the License at
 -
 -   http://www.apache.org/licenses/LICENSE-2.0
 -
 - Unless required by applicable law or agreed to in writing, software
 - distributed under the License is distributed on an "AS IS" BASIS,
 - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 - See the License for the specific language governing permissions and
 - limitations under the License.
 -->

# Catalog-managed metadata plan

## Status

This document is an implementation and validation plan, not a compatibility commitment. The plan
is intended to guide an experimental implementation and identify the contracts that must be proven
before catalog-managed metadata can become a practical alternative to file-managed metadata.

## Objective

Enable catalogs to store Iceberg metadata as database records rather than metadata files, while
keeping data and delete files in object storage. A catalog-managed table should be loadable,
committable, and plannable without writing or reading table metadata JSON, manifest lists, or
manifests.

The intended outcome is not to put serialized metadata files in database binary columns. The
catalog database is authoritative and stores the logical contents of table metadata, snapshots,
manifests, and manifest entries in a representation suitable for transactional updates and indexed
scan planning.

The implementation must preserve Iceberg semantics and interoperability. It must address format
evolution, concurrent commits, deletes, branches and tags, incremental scans, metadata tables,
maintenance, garbage collection, and recovery. Performance must be demonstrated against
file-managed metadata under realistic storage and database conditions.

## Design principles

1. **Separate semantics from representation.** Iceberg defines table, commit, and scan semantics.
   Each catalog owns its physical database schema and indexing strategy.
2. **Keep database details behind a service boundary.** REST is the preferred client boundary so
   engines do not require the catalog database schema, driver, or credentials.
3. **Use logical records, not file blobs.** JSON and Avro blobs may be accepted temporarily for
   import, but are not the authoritative steady-state representation.
4. **Maintain one authority.** Catalog-managed metadata is authoritative. Optional metadata-file
   materialization is an export for compatibility, not a second commit path.
5. **Preserve file-managed behavior by default.** Existing catalogs and clients continue to use
   metadata files unless catalog-managed metadata is explicitly negotiated.
6. **Prove contracts before stabilizing APIs.** Prototype catalog behavior first and promote only
   the smallest reusable contracts into public APIs.
7. **Measure end-to-end behavior.** Compare client-visible latency, throughput, resource use, and
   operational cost rather than isolated database queries.

## Ownership boundaries

### Iceberg library and REST protocol

Iceberg should define:

- capability negotiation for catalog-managed metadata;
- an opaque committed revision identity that does not require a metadata file location;
- logical commit requirements and updates;
- transport for file additions, removals, delete metadata, and statistics that a catalog can ingest
  without retaining manifests;
- server-side planning requests and scan-task responses;
- fallback, compatibility, and failure behavior;
- conformance tests that apply to any catalog-managed implementation.

Iceberg core must not define SQL tables or depend on a database implementation.

### Catalog implementation

A catalog implementation should own:

- relational layout, indexes, partitioning, and migration of its database schema;
- transaction isolation and revision allocation;
- translation of logical Iceberg updates into database mutations;
- scan expression translation and query planning;
- retention, reachability, and physical deletion of metadata records;
- optional import and materialization of standard Iceberg metadata files;
- database-specific operational guidance and tuning.

## Proposed operating modes

Authority and compatibility materialization are independent settings:

```text
metadata-management = files | catalog
metadata-file-materialization = none | table | manifest-list | all
```

These names are provisional. They should initially be catalog configuration, not table properties.
A catalog may later support per-table selection if it can guarantee that the mode is immutable or
safely migrated.

When `metadata-management=catalog`, database records are authoritative in every materialization
mode. Materialized files must identify the catalog revision from which they were produced and must
never be accepted as an independent concurrent commit.

## Architecture

The first prototype should be a row-native catalog behind the Iceberg REST protocol:

```text
engine
  |-- logical commit and validation requirements --> REST catalog
  |<------------- committed revision --------------|
  |                                                  |-- relational metadata
  |--------------- plan scan ---------------------->|
  |<---------------- scan tasks --------------------|
  |-- read selected data and delete files ------------> object storage
```

The prototype should not begin by changing `JdbcCatalog`. `JdbcCatalog` is a client-side catalog
whose database entry points to a metadata file. Building behind REST avoids exposing a physical
schema as a client API and provides a place to perform server-side planning.

## Logical data model

The prototype schema should represent at least the following logical entities. Names and
normalization are implementation details rather than proposed standard SQL names.

- catalogs, namespaces, and tables;
- immutable table revisions and their parent revisions;
- table properties and table locations;
- schemas, fields, identifier fields, and schema history;
- partition specs, partition fields, and spec history;
- sort orders, sort fields, and sort-order history;
- snapshots, parents, sequence numbers, summaries, and timestamps;
- branches, tags, retention policies, and snapshot reference history;
- data files, delete files, deletion vectors, and file content metadata;
- snapshot-to-file changes, including added and deleted status;
- typed partition values;
- per-column counts, bounds, null counts, NaN counts, and split offsets;
- key metadata and encryption metadata;
- statistics and partition-statistics references;
- commit attempts, idempotency keys, and optional audit information;
- materialization state when compatibility files are enabled.

The schema should avoid copying the complete live file set for each snapshot. Candidate models must
be tested, including append-only change records, persistent set structures, periodic checkpoints,
and database-native temporal representations.

## Revision and concurrency model

File-managed Iceberg commonly uses a metadata location as both content pointer and version token.
Catalog-managed metadata needs an opaque revision identity with these properties:

- unique and immutable within a table;
- returned on load and commit;
- usable as an HTTP validator and cache key;
- included in commit requirements for compare-and-swap behavior;
- stable across retries of the same successfully applied request;
- unrelated to a database transaction ID that may be recycled or reveal implementation details;
- able to identify materialized compatibility files;
- sufficient to distinguish create, replace, drop, and recreate lifetimes.

Commits must atomically validate requirements, apply updates, create the new revision, and advance
the table pointer. A timeout after submission must result in either a recoverable idempotent retry or
an unknown commit state with a supported status lookup. Successful commits must not report failure
because of nonessential work such as asynchronous materialization.

The prototype must exercise serializable isolation and explicit compare-and-swap implementations.
It should document anomalies prevented by each supported isolation level.

## Commit ingestion

Table-level REST metadata updates already provide a useful logical commit vocabulary, but file
changes require an additional scalable ingestion path. The prototype should evaluate:

1. structured file-entry batches sent through REST;
2. database-native bulk load into commit-scoped staging records;
3. staged standard manifests that the server imports and then deletes or retains only as
   compatibility output;
4. server-side manifest construction from writer completion messages.

The steady-state protocol must support millions of file changes without one JSON request, must be
idempotent, and must checksum or otherwise validate every batch. Staged batches must be invisible
until the final commit and safely reclaimable after abandonment.

Writers must not upload database credentials or write directly into authoritative catalog tables.
Database-native bulk loading may be an implementation optimization behind a catalog-controlled
endpoint.

## Scan planning

The relational planner must match client-side Iceberg planning for:

- snapshot and reference selection;
- partition projection across spec evolution;
- inclusive and strict metrics evaluation;
- case sensitivity and field-ID-based binding;
- residual expression calculation;
- data and delete file association;
- equality delete applicability and sequence numbers;
- position deletes and deletion vectors;
- split planning and locality information;
- encryption metadata;
- incremental append and changelog scans;
- metadata tables;
- table format versions and feature gating.

Expression translation must use bound parameters and preserve Iceberg's null, NaN, binary, decimal,
timestamp, and string comparison semantics. Unsupported expressions must either produce a correct
conservative plan with residuals or fail before returning incomplete results.

Planning responses must stream or paginate. Cancellation, deadlines, maximum result sizes, and
backpressure must be defined. A plan must be consistent with one committed revision even while
newer commits complete.

## Correctness and lifecycle corner cases

The implementation is not practical until it handles the following areas.

### Schema, partition, and sort evolution

- Preserve stable field IDs and historical definitions.
- Plan every file with the spec and sort order under which it was written.
- Reject ID reuse and invalid update sequences exactly as file-managed Iceberg does.
- Retain definitions while any reachable snapshot needs them.

### Deletes and sequence numbers

- Preserve data and file sequence-number inheritance rules.
- Apply equality deletes only to eligible data files.
- Match position deletes and deletion vectors without losing referenced-file semantics.
- Handle rewritten files and delete-file removal atomically.

### Snapshot references and time travel

- Support branches, tags, fast-forward, cherry-pick, rollback, and retention policies.
- Make time-based lookup deterministic when commits have equal timestamps.
- Prevent expiration of snapshots reachable through retained references.

### Maintenance

- Implement snapshot expiration as a transactional reachability change.
- Separate logical removal from asynchronous physical metadata reclamation.
- Identify data and delete files that become unreachable without deleting files still reachable from
  another revision or reference.
- Support orphan staging-record cleanup and materialized-file cleanup.
- Define catalog behavior for rewrite manifests, rewrite data files, and rewrite position deletes
  when there are no authoritative manifests.

### Failure and recovery

- Retry request batches without duplicate file records or snapshots.
- Resolve commit state following connection loss.
- Recover from catalog process failure between staging and pointer advancement.
- Recover or roll back database migrations.
- Treat materialization and notification failures as post-commit work.
- Verify backups restore a mutually consistent catalog revision and object-store file set.

### Import, export, and interoperability

- Import an existing table while validating the complete reachable metadata graph.
- Record an import watermark so concurrent source commits cannot be silently lost.
- Export a selected revision as valid standard Iceberg metadata files.
- Define whether exported files are read-only snapshots or can be registered elsewhere.
- Provide an explicit migration workflow between file and catalog authority.
- Ensure clients that lack the capability fail with an actionable error rather than treating an
  absent metadata location as table corruption.

### Security and tenancy

- Authorize planning and commit operations at the catalog boundary.
- Do not expose database credentials to engines.
- Validate locations before returning or materializing them.
- Use parameterized SQL for expressions and metadata values.
- Bound planning cost and ingestion sizes to prevent resource exhaustion.
- Preserve table and namespace isolation in every query and bulk operation.
- Audit administrative import, migration, and materialization actions.

## Performance validation

No performance claim should be made without an end-to-end comparison against file-managed Iceberg.
Benchmarks should report distributions and resources for both the client and service.

### Workloads

- cold and warm table load;
- full-table planning;
- selective partition and metrics pruning;
- many-small-manifest and few-large-manifest layouts;
- metadata tables and incremental scans;
- append, overwrite, row-delta, rewrite, and snapshot-expiration commits;
- concurrent readers and writers;
- conflicting and non-conflicting commits;
- tables ranging from thousands to tens of millions of files;
- long histories and many branches and tags.

### Measurements

- p50, p95, and p99 client-visible latency;
- planning throughput and time to first task;
- request count and bytes transferred;
- catalog and client CPU and memory;
- database query count, rows examined, cache hit rate, and temporary space;
- database rows, index bytes, write-ahead log bytes, and replication lag per commit;
- object-store requests and bytes;
- conflict, retry, timeout, and unknown-state rates;
- metadata retention and vacuum cost.

Tests must include realistic object-store latency and a remote database connection. Local files
versus an embedded database are useful for profiling but cannot substantiate production claims.

### Advancement gates

Before proposing stable support, the prototype must:

- pass differential correctness tests against file-managed planning and commits;
- demonstrate a material improvement in cold or selective scan planning at representative scale;
- avoid unacceptable regression for large append and rewrite commits;
- show bounded database growth with documented retention and reclamation;
- sustain concurrent workloads without inconsistent plans or excessive contention;
- complete import, export, backup, restore, and upgrade exercises.

Exact numeric thresholds should be set before benchmark runs and recorded with the benchmark
environment.

## Delivery phases

### Phase 0: contracts and benchmark baseline

- Inventory every metadata-file-location assumption in load, commit, cache, maintenance, and tests.
- Capture file-managed REST load, plan, and commit baselines.
- Define revision and capability semantics without changing existing defaults.
- Build a differential test harness that compares scan tasks and reachable snapshots.

### Phase 1: table state in rows

- Implement an experimental REST catalog with relational schemas, specs, sort orders, properties,
  snapshots, and references.
- Apply existing logical REST updates transactionally.
- Load committed metadata using an opaque revision rather than a metadata file.
- Continue using standard manifests during this phase.

### Phase 2: row-native append and planning

- Add staged, idempotent bulk ingestion of data-file entries.
- Store snapshot file changes in rows without authoritative manifests.
- Plan unpartitioned and identity-partitioned append-only tables from SQL.
- Compare correctness and performance with standard manifests.

### Phase 3: complete scan semantics

- Add spec evolution, metrics pruning, residuals, splits, delete files, deletion vectors, and
  encryption metadata.
- Add incremental scans and metadata tables.
- Add streaming, pagination, cancellation, and revision-consistent plans.

### Phase 4: complete commit and maintenance semantics

- Add overwrite, row delta, rewrites, branches, tags, expiration, and garbage collection.
- Add idempotent recovery and commit-status lookup.
- Validate high-volume commits and concurrent writers.

### Phase 5: interoperability and operations

- Add import, optional metadata-file materialization, and export.
- Exercise migration in both directions.
- Document backup, restore, replication, schema migration, and disaster recovery.
- Run compatibility suites across supported engines.

### Phase 6: stabilization

- Review experimental APIs and retain only representation-neutral contracts.
- Add catalog-managed conformance tests.
- Document compatibility guarantees and unsupported client behavior.
- Propose specification or REST changes separately where interoperability requires them.

## Initial repository work

The first implementation change should be a narrow, test-only prototype rather than a new public
SPI. It should:

1. use the REST server test infrastructure and an embedded relational database;
2. introduce an internal opaque revision for table load and commit;
3. store table-level state in rows while leaving manifests in files;
4. add differential tests against the existing REST catalog behavior;
5. add benchmarks before moving manifest entries into rows.

Once table-level behavior is correct, row-native append ingestion and planning can be added behind
an experimental REST capability. Findings from that implementation should determine the production
API rather than making the prototype schema public.

## Open decisions

- Whether a revision belongs in the existing metadata-location field as an opaque URI or requires a
  separate REST field.
- Whether file-entry ingestion extends table commit or uses commit-scoped staging resources.
- Which database and deployment topology should be used for production-scale validation.
- Which snapshot file-membership representation gives acceptable read and write amplification.
- Whether compatibility materialization is synchronous, asynchronous, or on demand.
- How exported metadata records catalog authority and prevents accidental independent writes.
- Whether a catalog-managed table may switch authority in place or must be copied to migrate.
- Which administrative APIs expose retention, vacuum status, and materialization status.

These decisions should remain internal to the prototype until correctness tests and measurements
provide evidence for a reusable contract.
