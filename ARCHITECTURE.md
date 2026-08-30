# Rustvex Architecture — Convex-Lite on PostgreSQL

**Revision 3 · Simplified v1 implementation specification**
**Status:** proposed design. APIs and protocols below are requirements, not claims
that they are implemented.

Rustvex is a Rust-native logical document database built on PostgreSQL.

> **Rust owns database semantics. PostgreSQL owns durable transactional ordered
> storage.**

The v1 design deliberately favors a small correctness argument over maximum
write throughput:

- one active writer transaction per Rustvex database
- concurrent snapshot-consistent readers
- generic current-state document and index tables
- explicit ordered indexes
- commit-ordered revisions and durable invalidation records
- single-table realtime subscriptions
- blocked, resumable index backfills

The design does not attempt to reproduce all of Convex. It should first prove
that typed Rust documents, ordered byte indexes, atomic mutations, and reactive
queries work cleanly on PostgreSQL.

## Decisions at a glance

| Area | V1 decision |
|---|---|
| Physical storage | Five generic PostgreSQL tables; never one SQL table per logical model. |
| Writers | One writer at a time, using a PostgreSQL row lock acquired before application-data reads. |
| Revisions | Acquire the writer gate separately; increment the transactional counter only when first performing a document or catalog write. |
| Readers | One short `REPEATABLE READ READ ONLY` transaction per complete query. |
| Documents | Versioned envelope containing a named-field MessagePack map. |
| Indexes | Immutable physical generations with Rust-owned ordered keys; ordinary and unique indexes. |
| Queries | One logical table, one explicit index, and bounded range operations. |
| Changes | Consume complete bounded revisions using one completed-revision cursor. |
| Subscriptions | One statically known table per subscription; polling is durable truth and `NOTIFY` is only a wake-up. |
| Backfills | Block mutations to the affected table, build in bounded resumable batches, then activate atomically. |
| Schema evolution | Optional additions, explicit renames, index changes, and one typed maintenance migration path. |

The state row is a **write-contention point**, not a separate availability
component. PostgreSQL itself is the v1 availability boundary. The cost of the
writer gate must be measured and visible.

---

# 1. Goals, boundaries, and invariants

## 1.1 Architectural boundary

```text
Rustvex owns:
    logical schema and types
    generated validation and codecs
    document semantics
    logical query planning
    index definitions and ordered key encoding
    index maintenance and uniqueness
    revisions and change consumption
    subscriptions and schema deployment

PostgreSQL owns:
    durable bytes and WAL
    atomic transactions
    snapshots and row locks
    B-tree storage and byte comparisons
    ordered range scans
    crash recovery and physical maintenance
```

PostgreSQL sees table IDs, index IDs, document IDs, revisions, payload bytes, and
ordered index-key bytes. It does not need columns named `title`, `status`, or
`updated_at`.

One Rustvex database uses one isolated set of physical tables on one PostgreSQL
primary. Multiple Rustvex processes may connect to it.

## 1.2 V1 invariants

These are design tests, not aspirations:

1. PostgreSQL never interprets a logical document field.
2. A complete query observes one PostgreSQL snapshot.
3. Every Rustvex writer acquires the same gate before application-data reads.
4. One visible write transaction has one revision, published only after commit.
5. Documents and all active index entries change atomically.
6. Rustvex's versioned index codec alone defines logical index ordering.
7. Table and index identities do not change when declarations are reordered.
8. An obsolete schema binary cannot continue writing silently.
9. A subscription cannot miss a commit during initial execution or a rerun.
10. Persistent formats change only through an explicit version or new index
    generation.

## 1.3 Explicit non-goals

V1 does not include:

- application-level historical MVCC or audit history
- arbitrary revision snapshot reads
- concurrent writer transactions
- joins or multi-table query plans
- online write-through index backfills
- replica routing, automatic sharding, or distributed consensus
- full-text search or vector search
- offline mutation merging

PostgreSQL's own MVCC, WAL, vacuum, backup, and recovery remain part of the
system. “No custom MVCC” does not mean “no PostgreSQL MVCC.”

---

# 2. Logical schema and public model

## 2.1 Schema DSL

Keep the schema Rust-native and small:

```rust
schema! {
    users {
        name: String,
        email: String,
        age: Number?,
        nickname: Nullable<String>?,
        active: Bool,

        unique index by_email(email),
    }

    posts {
        title: String,
        body: String,
        author: Id<users>,
        published: Bool,
        created_at: Timestamp,

        index by_author(author),
        index by_published_time(published, created_at),
    }
}
```

An ordinary `index` permits duplicate logical keys. A `unique index` enforces
uniqueness when every indexed field is present and non-null. A unique index
does not create an entry when any component is missing or null. Consequently,
multiple documents may omit a uniquely indexed optional field.

Unique constraints are Rustvex semantics, not PostgreSQL constraints on
application fields. Under the writer gate, Rustvex performs an exact key lookup
before inserting a unique entry. Backfill fails with a structured duplicate
report if existing data violates the constraint.

## 2.2 Logical value model

| Type | Meaning |
|---|---|
| `String` | Valid UTF-8; bytewise ordering, with no locale normalization. |
| `Number` | Finite `f64`; negative zero is normalized to positive zero. |
| `Integer` | Signed 64-bit integer. |
| `Bool` | Boolean. |
| `Bytes` | Arbitrary byte string. |
| `Timestamp` | Signed 64-bit microseconds since the Unix epoch in UTC. |
| `Json` | JSON-compatible structured value; not indexable in v1. |
| `Id<table>` | A 16-byte identity statically associated with a logical table. |
| `T?` | Field may be missing; when present it is a non-null `T`. |
| `Nullable<T>` | Required field containing either `T` or explicit null. |
| `Nullable<T>?` | Field may be missing, null, or `T`. |

Missing and null are distinct. Missing means the field name is absent from the
document map. Null is an explicitly present value. JSON null inside a `Json`
value does not make the outer field missing.

The generic runtime representation is:

```rust
enum Value {
    Null,
    Bool(bool),
    Integer(i64),
    Number(f64),
    Timestamp(Timestamp),
    String(String),
    Bytes(Vec<u8>),
    Id(DocumentId),
    Json(serde_json::Value),
}
```

Missing is map absence rather than a `Value` variant. Index extraction can emit
an internal Missing sentinel for an absent ordinary-index field.

## 2.3 Generated code

`schema!` generates:

- a zero-sized marker type for each logical table
- a typed document containing read-only `id` and `revision` metadata
- typed insert, patch, and replacement inputs
- typed field, ordinary-index, and unique-index handles
- one static `CompiledTable` descriptor
- specialized validators, payload codecs, and index extractors

Illustrative internal types:

```rust
struct CompiledTable {
    name: &'static str,
    fields: &'static [CompiledField],
    indexes: &'static [CompiledIndex],
    schema_fingerprint: SchemaFingerprint,
}

struct TableBinding {
    table_id: TableId,
    schema_version: SchemaVersion,
    schema_generation: SchemaGeneration,
    indexes: Vec<IndexBinding>,
}
```

Compiled handles are symbolic. Runtime startup resolves them to stable persisted
IDs. IDs are never derived from declaration order. A schema fingerprint hashes
a canonical engine-owned descriptor, never Rust debug output or map iteration
order.

## 2.4 Document operations

V1 supports:

```rust
db.insert(users, values! { /* complete required input */ }).await?;
db.get(user_id).await?;
db.patch(user_id, patch! { /* Keep, Set, or Remove */ }).await?;
db.replace(user_id, values! { /* complete replacement */ }).await?;
db.delete(user_id).await?;
```

A patch distinguishes:

```rust
enum PatchField<T> {
    Keep,
    Set(T),
    Remove,
}
```

For nullable fields, `Set(Nullable::Null)` differs from `Remove`. Removing a
required field fails. Replacement omits optional fields and rejects omitted
required fields. Unknown fields are rejected unless contained inside a declared
`Json` field.

Typed IDs prevent using `Id<posts>` where `Id<users>` is required. References do
not imply existence checks or cascading deletes.

---

# 3. Physical storage and metadata

## 3.1 Bootstrap schema

Logical tables never become physical PostgreSQL tables:

```sql
CREATE TABLE rustvex_documents (
    table_id    INTEGER NOT NULL CHECK (table_id > 0),
    document_id BYTEA NOT NULL CHECK (octet_length(document_id) = 16),
    revision    BIGINT NOT NULL CHECK (revision > 0),
    payload     BYTEA NOT NULL CHECK (octet_length(payload) > 0),
    PRIMARY KEY (table_id, document_id)
);

CREATE TABLE rustvex_indexes (
    index_id    INTEGER NOT NULL CHECK (index_id > 0),
    key         BYTEA NOT NULL CHECK (octet_length(key) <= 2048),
    document_id BYTEA NOT NULL CHECK (octet_length(document_id) = 16),
    PRIMARY KEY (index_id, key, document_id)
);

CREATE TABLE rustvex_changes (
    revision    BIGINT NOT NULL CHECK (revision > 0),
    table_id    INTEGER NOT NULL CHECK (table_id > 0),
    document_id BYTEA NOT NULL CHECK (octet_length(document_id) = 16),
    operation   SMALLINT NOT NULL CHECK (operation IN (1, 2, 3)),
    PRIMARY KEY (revision, table_id, document_id)
);

CREATE TABLE rustvex_meta (
    key   TEXT PRIMARY KEY,
    value BYTEA NOT NULL
);

CREATE TABLE rustvex_state (
    singleton               SMALLINT PRIMARY KEY CHECK (singleton = 1),
    revision                BIGINT NOT NULL CHECK (revision >= 0),
    schema_generation       BIGINT NOT NULL CHECK (schema_generation > 0),
    pruned_through_revision BIGINT NOT NULL
        CHECK (pruned_through_revision >= 0),
    database_epoch          BYTEA NOT NULL
        CHECK (octet_length(database_epoch) = 16),
    storage_format_version  INTEGER NOT NULL
        CHECK (storage_format_version > 0),
    CHECK (pruned_through_revision <= revision)
);
```

Bootstrap creates all five tables, generates a random database epoch, inserts
the singleton state row, and initializes catalog metadata in one transaction
before serving requests.

The documents and indexes tables contain current state only. Deletion removes
the current document and its current index rows, then records a DELETE
invalidation. There are no application-level tombstones or historical payloads.

## 3.2 State-row responsibilities

The singleton row contains:

- the last committed visible revision
- the active catalog generation
- the complete-revision pruning watermark
- a database epoch that changes after restore/replacement
- the physical storage format version

A restored or replaced database must rotate `database_epoch` before serving
requests. This makes cursors from a rewound or unrelated history fail closed.

Locking and revision allocation are separate:

```sql
-- First database-state operation in every writer transaction.
SELECT revision, schema_generation, pruned_through_revision, database_epoch
FROM rustvex_state
WHERE singleton = 1
FOR UPDATE;
```

This acquires the global writer gate without changing visible state.

Only a transaction about to perform a document or catalog write increments the
revision:

```sql
UPDATE rustvex_state
SET revision = revision + 1
WHERE singleton = 1
  AND revision < 9223372036854775807
RETURNING revision;
```

Backfill batches, cleanup batches, and a handler that performs no effective CRUD
operation do not increment the revision. Schema activation, availability
changes, and document writes do. If later operations cancel an earlier write,
the already allocated revision may commit without document change rows.

## 3.3 Metadata format

Every `rustvex_meta.value` uses a small engine-owned envelope:

```text
magic:          4 bytes, ASCII RVM followed by 00
format_version: unsigned 16-bit big-endian integer
body:           one named-field MessagePack map
```

Metadata keys initially cover:

```text
catalog/active
catalog/table/<table_id>/schema/<schema_version>
allocation/next_table_id
allocation/next_index_id
maintenance/active
```

The active catalog is one bounded versioned blob. Maintenance progress is a
separate bounded record so a checkpoint does not rewrite the complete catalog.
Metadata decoding rejects unknown versions, duplicate fields, trailing bytes,
and values above the configured metadata limit.

Catalog or job replacement is transactional. Do not implement a general
metadata database inside `rustvex_meta`.

## 3.4 Stable identities

Persist positive `TableId` and `IndexId` values. Never calculate identity from
declaration order and never recycle retired IDs.

A logical index name points to one active immutable physical generation:

```text
same definition and encoder version -> reuse index_id
changed fields, kind, or encoding    -> allocate replacement index_id
```

Renames are explicit catalog operations that preserve the chosen identity. A
same-named declaration is not automatically assumed to be a rename.

---

# 4. Persistent encodings

## 4.1 Document envelope

Documents use:

```text
magic:          4 bytes, ASCII RVX followed by 00
format_version: unsigned 16-bit big-endian integer, initially 1
schema_version: unsigned 64-bit big-endian integer
body:           one named-field MessagePack map
```

The BYTEA length is the envelope boundary. Reject trailing bytes after the map.
Generated codecs preserve integers, finite numbers, bytes, timestamps, typed
IDs, missing fields, and null.

Named fields permit declaration reordering and compatible optional additions.
`rmp-serde::to_vec_named` is an implementation tool, not the compatibility
contract. Rustvex still defines duplicate handling, missing/null semantics,
special logical types, limits, and supported historical schema versions.

New writers emit the active schema version. Decoders support only explicitly
registered historical versions. Persistent bytes never depend on Rust memory
layout.

## 4.2 Ordered index values

Document encoding and index encoding are separate formats. Index encoder v1
orders values as:

```text
Missing < Null < Bool < Integer < Number < Timestamp < String < Bytes < Id
```

Suggested tags:

```text
0x01 Missing      0x02 Null        0x10 Bool
0x20 Integer      0x30 Number      0x40 Timestamp
0x50 String       0x60 Bytes       0x70 Id
```

Values of different types follow tag order; there is no cross-type numeric
coercion. `Json` is not indexable.

Signed integers and timestamps flip the sign bit and use big-endian bytes:

```rust
fn sortable_i64(value: i64) -> [u8; 8] {
    ((value as u64) ^ 0x8000_0000_0000_0000).to_be_bytes()
}
```

Numbers must be finite. Normalize both input zeros to positive zero, then use
the sign-aware IEEE-754 transform:

```rust
fn sortable_f64(value: f64) -> Result<[u8; 8], NumberError> {
    if !value.is_finite() {
        return Err(NumberError::NotFinite);
    }
    let bits = if value == 0.0 { 0.0 } else { value }.to_bits();
    let ordered = if bits & (1_u64 << 63) != 0 {
        !bits
    } else {
        bits ^ (1_u64 << 63)
    };
    Ok(ordered.to_be_bytes())
}
```

Strings encode their UTF-8 bytes; Bytes use their raw bytes. Both escape:

```text
embedded 00 -> 00 FF
component end -> 00 00
```

An indexed `Id<T>` encodes its tag, the table ID as an unsigned four-byte
big-endian integer, then its 16 identity bytes.

Compound keys concatenate complete components. The physical primary key adds
the separate document-ID tie-breaker:

```text
(index_id, encode(field_1, field_2, ...), document_id)
```

## 4.3 Range boundaries

`prefix_successor(bytes)` increments the rightmost non-`FF` byte and truncates
the suffix. If no byte can be incremented, the result is an unbounded upper
endpoint.

For equality prefix `p` and ranged field `x`:

```text
x >= v -> lower = encode(p, v)
x >  v -> lower = prefix_successor(encode(p, v))
x <  v -> upper = encode(p, v)
x <= v -> upper = prefix_successor(encode(p, v))
```

Intersect those endpoints with the complete equality-prefix interval. Generate
separate SQL statement shapes for unbounded endpoints rather than inventing
minimum or maximum byte strings.

## 4.4 Encoding compatibility

Required property:

```text
logical_compare(A, B) == byte_compare(encode(A), encode(B))
```

Property tests compare database range results with an independent logical
reference filter. Golden vectors freeze every encoder version. Changing type
order, escaping, scalar transformations, or tuple semantics requires a new
immutable index generation and backfill.

---

# 5. Queries and read consistency

## 5.1 Query surface

Every v1 indexed query targets one logical table and one explicit index:

```rust
db.query(posts)
    .with_index(posts.by_published_time, |q| {
        q.eq(posts.published, true)
         .gte(posts.created_at, start)
    })
    .descending()
    .take(50)
    .await?;
```

V1 supports:

- get by typed document ID
- `eq`, `gt`, `gte`, `lt`, and `lte`
- ascending or descending traversal of the entire compound order
- `first`, bounded `take`, and bounded `collect`
- deterministic keyset pagination

For index `[a, b, c]`, accept zero or more leading equalities followed by at
most one ranged field. Do not silently turn a skipped-prefix condition into an
unbounded full scan.

There are no post-index filters in v1. This keeps limits, cursors, and
subscriptions predictable.

## 5.2 Snapshot protocol

A read context uses one pinned connection and one short transaction. It may run
several ID lookups or single-table indexed queries within one consistent
application read operation, but it does not provide joins or multi-table query
planning. Its budgets are cumulative across all reads in the context.

The transaction begins:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY;

SELECT revision, schema_generation, pruned_through_revision, database_epoch
FROM rustvex_state
WHERE singleton = 1;
```

That first ordinary read establishes the snapshot. In the same transaction:

1. load or verify catalog bindings for the observed generation
2. confirm the table and immutable index generation are readable
3. compile typed bounds
4. scan ordered index entries
5. bulk-fetch documents
6. decode and restore index order
7. commit and release the snapshot

Example index scan:

```sql
SELECT key, document_id
FROM rustvex_indexes
WHERE index_id = $1
  AND key >= $2
  AND key < $3
ORDER BY key ASC, document_id ASC
LIMIT $4;
```

Fetch those documents on the same transaction:

```sql
SELECT document_id, payload, revision
FROM rustvex_documents
WHERE table_id = $1
  AND document_id = ANY($2::bytea[]);
```

`ANY` does not define output order, so Rustvex restores the scan order. A
dangling row in a READY index is an integrity error, not a silently omitted
document.

The result carries its observed revision and schema generation for internal
subscription coordination. It does not create a historical snapshot that can
be reopened after the transaction.

Mutation reads reuse the existing mutation transaction and do not open a
separate read-only snapshot.

## 5.3 Query cursors

The public API returns an opaque cursor token containing:

```text
cursor format version
database epoch
immutable index_id and encoder version
normalized query fingerprint and direction
last key and document_id
```

Continuation uses:

```sql
-- Ascending
AND (key, document_id) > ($last_key, $last_document_id)
ORDER BY key ASC, document_id ASC

-- Descending
AND (key, document_id) < ($last_key, $last_document_id)
ORDER BY key DESC, document_id DESC
```

The adapter validates the cursor against the current request and original query
bounds. A modified cursor can at most change position inside those recomputed
bounds. Integrity protection can be added if future cursor fields affect trust.

Pagination is deterministic keyset pagination, not a frozen snapshot across
requests. Concurrent index-key changes can cause repeats or skips between pages.
Historical pagination is a separate feature.

---

# 6. Mutations, uniqueness, and revisions

## 6.1 Mutation transaction

Every mutation begins before any application-data read:

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;

SELECT revision, schema_generation, pruned_through_revision, database_epoch
FROM rustvex_state
WHERE singleton = 1
FOR UPDATE;
```

After acquiring the gate:

1. verify compiled schema compatibility and table availability
2. run all document reads through this transaction
3. apply and validate logical changes
4. allocate revision `R` on the first effective visible write
5. write documents at `R` and maintain changed index tuples
6. coalesce one final change record per touched document
7. optionally call `pg_notify` inside the transaction
8. commit

Later handler reads see earlier SQL writes in the same transaction. No custom
pending-index overlay is needed.

Complete each CRUD operation's document and index maintenance before the next
handler operation. Any validation or storage error poisons the v1 mutation and
rolls back the whole transaction. Nested savepoint recovery is deferred.

The handler must not make network calls, wait for user input, or perform
irreversible external side effects while holding the gate. Enforce mutation,
lock-wait, and SQL-statement deadlines.

## 6.2 Index maintenance

For an update:

```text
read and decode old document
apply change and validate new document
encode new payload
calculate old and new key for every active index
delete only stale tuples
insert only new tuples
update document revision and payload
```

Unchanged keys remain in place. A delete regenerates keys from the old document,
removes those exact tuples, deletes the document, and appends a DELETE change.
No per-document index manifest is required.

## 6.3 Unique indexes

Before inserting a fully present, non-null unique key:

```sql
SELECT document_id
FROM rustvex_indexes
WHERE index_id = $1 AND key = $2
LIMIT 1;
```

If another document owns the key, return a structured uniqueness error. The
global writer gate prevents a second Rustvex writer from passing the same check
concurrently.

Uniqueness is checked after each CRUD operation, using that transaction's
current SQL-visible state. Delete-then-insert reuse works, but directly swapping
two occupied unique keys is not supported in v1; it requires an intermediate
free key or separate operations. This keeps uniqueness compatible with the
simple read-your-writes mutation model.

The physical index table does not independently enforce logical uniqueness.
Correctness therefore requires the runtime role to be the only writer. A future
concurrent-writer design must add a PostgreSQL-enforced or lock-based uniqueness
protocol before removing the gate.

## 6.4 Revision semantics

A revision identifies one committed visible Rustvex transaction. Every document
and change record produced by it uses the same revision.

The increment is transactional and provisional until commit. Rollback removes
the increment together with document, index, and change writes. Do not publish
`R` before commit is confirmed.

A handler that performs no effective CRUD operation does not increment the
revision. A maintenance batch that changes only unreachable build state does
not increment it. Catalog activation, mutation availability changes, and
document writes do. If a later operation cancels an earlier write—for example,
insert followed by delete—the allocated revision may commit without a document
change record. Whole-revision polling already handles that case.

Do not use a PostgreSQL sequence for this cursor. Under the writer gate, one
transaction holds the state-row lock through commit, so a lower committed
revision cannot appear after a higher revision.

Clients must not rely on gapless numbers. Storage upgrades, recovery, or future
protocols may introduce gaps.

## 6.5 Coalesced document changes

Produce at most one change record per document:

| Before mutation | Final state | Record |
|---|---|---|
| Absent | Present | INSERT |
| Present | Present and written | UPDATE |
| Present | Absent | DELETE |
| Absent | Absent | None |

Insert→patch is one INSERT. Patch→delete is one DELETE. Insert→delete emits no
document record. A no-op patch can skip all physical writes.

Enforce `MAX_CHANGED_DOCUMENTS_PER_REVISION = 1_000` before commit. This is a
protocol invariant used by whole-revision change consumption, not merely a
performance default.

A connection failure during COMMIT can make the outcome unknown. Return
`CommitOutcomeUnknown` and do not automatically replay a non-idempotent
mutation. Durable request deduplication is deferred.

---

# 7. Change consumption and subscriptions

## 7.1 Change log

`rustvex_changes` is a retained invalidation log:

```text
1 = INSERT
2 = UPDATE
3 = DELETE
```

It does not contain old payloads, new payloads, intermediate operations, or an
audit history. Fetching current state after revision 100 may return a document
already updated at revision 105.

Optional `LISTEN/NOTIFY` only wakes a consumer. Issue `NOTIFY` inside the
writing transaction so delivery occurs after commit. Periodic polling remains
the recovery path.

## 7.2 Whole-revision polling

The internal cursor is deliberately small:

```rust
struct ChangeCursor {
    database_epoch: DatabaseEpoch,
    completed_revision: Revision,
}
```

For each poll cycle, open a short `REPEATABLE READ READ ONLY` transaction and
read state watermark `H`, catalog generation `G`, pruning watermark `P`, and
database epoch `E`.

Reject/reset the cursor when its epoch differs or
`completed_revision < P`. Otherwise find the next complete revision:

```sql
SELECT revision
FROM rustvex_changes
WHERE revision > $1
  AND revision <= $2
ORDER BY revision
LIMIT 1;
```

If revision `N` exists, fetch all of it:

```sql
SELECT table_id, document_id, operation
FROM rustvex_changes
WHERE revision = $1
ORDER BY table_id, document_id;
```

At most 1,000 rows can exist because mutation size is a hard protocol limit.
Hand the complete revision to the local dispatcher, then advance the cursor to
`N`.

If no change revision remains through `H`, advance directly to `H`. This handles
catalog-only and other visible revisions with no document rows. Check `G` even
when the change query returns nothing.

Delivery may be duplicated after local failure; invalidation is idempotent.
Exactly-once external event delivery is not promised.

## 7.3 Retention

`pruned_through_revision = P` means all change rows at revisions `<= P` may be
absent. Prune complete revisions only:

```text
acquire writer gate without incrementing revision
delete rows through selected completed revision P'
set pruned_through_revision = P'
commit atomically
```

Retention validation and page reads occur in one snapshot. A stale consumer
does a full fresh subscription query instead of pretending it caught up.

The initial release may ship with pruning disabled by default. Production
deployments can enable bounded pruning only after configuring a retention
window large enough for expected process outages.

## 7.4 Single-table subscriptions

Every v1 query targets one logical table, so its dependency is statically known.
Do not implement dynamic dependency sets or temporarily watch every table.

A serialized local dispatcher prevents registration races:

```text
1. register watcher for table_id
2. begin retaining its pending revision
3. execute initial query in a fresh read snapshot
4. publish result + observed revision + schema generation
5. rerun if a pending relevant revision is newer
```

Track:

```text
table_id
last_published_revision
highest_pending_revision
running
subscription_generation
reset_required
```

An invalidation arriving during a rerun must remain pending. After completion,
run again when `highest_pending_revision` exceeds the result's observed
revision. Coalescing redundant invalidations is allowed.

Schema-generation changes rebind or reset affected subscriptions. Results are
individually snapshot-consistent, but multiple independent subscriptions do not
transition atomically as a group. V1 does not promise every intermediate state
or an offline client replica.

---

# 8. Schema deployment and index backfills

## 8.1 Catalog generation

Startup compares compiled schema fingerprints with the persisted active catalog.
An old process must never automatically redeploy its own schema over a newer
catalog.

Increment `schema_generation` whenever active field/index bindings or logical
table availability changes. Backfill checkpoints do not increment it.

Every mutation checks the generation after acquiring the writer gate. Every
query checks it inside its snapshot. A cached binding is valid only for the
generation that produced it.

```text
generation changed, same compiled definitions:
    reload and rebind

generation changed, incompatible compiled definitions:
    return SchemaChanged
```

## 8.2 Supported schema changes

V1 directly supports:

- adding an optional field
- explicitly renaming a table or field while preserving identity
- adding ordinary or unique indexes
- retiring an index
- rejecting an incompatible compiled schema safely

Adding an optional field updates logical metadata and retains the historical
payload decoder. It does not alter a PostgreSQL application table.

Required-field additions, type changes, and destructive field changes require
one application-supplied typed maintenance migration:

```text
block reads and mutations for the target table when required
scan documents in bounded batches
invoke one typed migration function
validate and write through the normal engine
checkpoint progress
validate all current documents
activate the new schema
```

V1 does not include a general migration language, automatic migration planner,
or zero-downtime incompatible migration.

Table removal is an explicit destructive administrative operation. Mark the
table unavailable, invalidate subscriptions through a generation change, and
clean rows in bounded batches. Never recycle its identity.

## 8.3 Index states

Use four immutable-generation states:

```text
Building -> Ready
         -> Failed
Ready    -> Retired
```

Only Ready indexes serve normal reads and writes. Retired generations are
unreachable from the active catalog and can be cleaned in bounded batches.
A failed generation is never redefined in place.

## 8.4 Backfill protocol

Only one schema/index maintenance job may exist per Rustvex database. A worker
uses a dedicated PostgreSQL connection and holds a session advisory lock for
the job's lifetime. Connection loss releases the lock; a resumed worker
reacquires it.

The persistent job contains:

```text
job_id
index_id and owning table_id
expected schema generation
document_id checkpoint
state and failure details
```

### Prepare

In one writer-gated visible transaction:

1. verify there is no active maintenance job
2. allocate the immutable index ID
3. persist the Building generation and job
4. mark the owning table `MUTATIONS_BLOCKED`
5. increment revision and schema generation
6. commit

Existing Ready indexes remain readable. A mutation attempting to access the
blocked table fails quickly. Mutations touching other tables may run between
backfill batches.

Before preparation, the maintenance command reports document count, estimated
payload bytes, selected batch limits, and any detected obvious limit problems.
The availability cost must be visible to the operator.

### Build

Each bounded batch opens a new transaction, acquires the ordinary writer gate
without incrementing the revision, and verifies the advisory lock, job ID, and
expected generation:

```sql
SELECT document_id, payload, revision
FROM rustvex_documents
WHERE table_id = $1
  AND document_id > $2
ORDER BY document_id
LIMIT $3;
```

Use a separate first-page statement. Decode each document, validate and encode
its new key, insert with `ON CONFLICT DO NOTHING`, and update the checkpoint in
the same transaction.

For a unique index, skip documents with any missing/null component and fail the
job if two present keys are equal.

Because mutations to the table remain blocked across batches, the scan is
stable. The gate is released after each batch so other-table mutations can
proceed.

### Activate or abort

A final gated visible transaction verifies end-of-scan and the expected job,
switches the generation to Ready, removes the table block and job, increments
revision and schema generation, then commits.

On a bad document, duplicate unique key, or oversized encoded key, mark the job
Failed and retain the table block until explicit resume or abort. Abort retires
the incomplete generation, removes the block, clears the job, and increments
revision and schema generation atomically. Partial rows are cleaned later.

Crash recovery resumes the last committed checkpoint. The advisory lock plus
job-ID checks replace a more elaborate fencing-token protocol in v1.

---

# 9. Limits, operations, and observability

## 9.1 Hard protocol and format limits

These are v1 invariants:

```text
document ID:                         16 bytes
encoded logical index key:           2,048 bytes
fields per compound index:           16
changed documents per revision:      1,000
Number:                              finite f64, normalized zero
Timestamp:                           signed i64 UTC microseconds
```

The 2,048-byte key limit includes tags and escaping. Enforce it in Rust and with
the physical CHECK constraint. Test maximum-size composite entries against
every supported PostgreSQL major version using the supported standard page
size. It is a Rustvex compatibility limit, not a universal PostgreSQL limit.

## 9.2 Configurable operational budgets

Proposed defaults:

```text
encoded document payload:            1 MiB
metadata value:                       1 MiB
query candidate documents:           10,000
query decoded payload bytes:         16 MiB
mutation encoded payload bytes:      16 MiB
backfill batch:                       500 documents
```

Backfill batches are additionally byte- and time-bounded. Decoders cap nesting,
container elements, and allocations before trusting encoded lengths.

A complete query exceeding a budget fails with a structured error; it is never
silently returned as a complete truncated result. Deployment configuration may
lower operational defaults. Raising them requires explicit testing.

## 9.3 Timeouts and cancellation

Set finite:

- connection-acquisition timeout
- writer-gate lock timeout
- statement timeout
- read-snapshot duration
- mutation-handler duration
- maintenance-batch duration

Cancellation rolls back or discards the connection. A transaction-in-progress
connection must never return to the pool.

## 9.4 Required instrumentation

Expose at least:

```text
writer_gate_wait_seconds
mutation_transaction_seconds
read_snapshot_seconds
change_consumer_lag_revisions
change_log_rows
subscription_reruns
subscription_coalesced_invalidations
backfill_documents_processed
backfill_batch_seconds
payload_bytes
index_key_bytes
postgres_round_trips
```

Log table/index/revision context without payload contents or credentials.
Provide health output for catalog generation, storage version, current revision,
consumer lag, active maintenance job, and table availability.

## 9.5 Performance contract

Do not claim a write-throughput advantage over direct PostgreSQL or Convex.
Benchmark:

- ID lookups
- ordered top-N scans
- inserts and deletes
- indexed and non-indexed patches
- multi-document mutations
- writer-gate wait time under contention
- subscription fan-out and rerun rate
- backfill throughput
- WAL volume, dead tuples, and storage growth

Measure p50, p95, and p99 latency plus throughput and database round trips. If
the writer gate misses the product's measured workload target, concurrent
writers require a new transaction and commit-publication design; per-table
locks or a sequence alone are not safe drop-in replacements.

---

# 10. V1 product scope and implementation

## 10.1 Usable v1 feature set

```text
Schema:
    schema!, values!, patch!
    required, optional, and nullable fields
    typed IDs, fields, ordinary indexes, and unique indexes

Documents:
    insert, get, patch, replace, delete
    versioned named-field payloads

Queries:
    one table and one explicit index
    equality and range operations
    ascending/descending keyset pagination
    finite collection budgets

Transactions:
    atomic multi-document mutations
    one writer at a time
    concurrent snapshot-consistent readers
    read-your-writes

Revisions:
    transactional visible revisions
    complete bounded revision consumption

Realtime:
    single-table subscriptions
    registration-safe and rerun-safe invalidation
    polling plus optional NOTIFY wake-ups

Schema:
    stable identities and stale-binary fencing
    compatible optional-field additions
    typed blocked migrations
    resumable blocked index backfills
```

This is enough for CRUD SaaS applications, collaborative tools, dashboards,
internal tools, notes applications, and other modest-write realtime products.
Its documented limitations are serialized write throughput and mutation
availability during target-table maintenance.

## 10.2 Implementation order

1. Workspace, physical bootstrap, metadata envelope, IDs, and errors.
2. Logical values, document envelope, ordered index codec, and property tests.
3. `schema!`, `values!`, `patch!`, generated documents, and compile-fail tests.
4. Writer gate, CRUD, revision allocation, index maintenance, and uniqueness.
5. Snapshot reads, planner, range scans, ordering, budgets, and cursors.
6. Complete-revision polling and single-table subscriptions.
7. Catalog generation, compatible schema deployment, and blocked backfills.
8. Typed maintenance migrations, retention, operational tooling, and benchmarks.

Each stage ends in an end-to-end vertical slice. Do not start distributed,
historical, or concurrent-writer work before the acceptance suite passes.

---

# Appendix A. Required acceptance tests

These tests must run against real PostgreSQL where transaction or B-tree
behavior matters.

| Test | Required outcome |
|---|---|
| Cross-process writer gate | Writer B cannot read application state until writer A commits or rolls back. |
| Gate without revision | No-op mutation, backfill batch, and cleanup batch do not advance revision. |
| Visible revision | First effective document/catalog write allocates one revision shared by all its rows. |
| Rollback | Counter, documents, indexes, and changes all return to prior state. |
| Commit order | A lower committed revision cannot become visible after a consumed higher revision. |
| Read-your-writes | Mutation reads observe its earlier inserts, patches, index moves, and deletes. |
| Snapshot race | A query paused between index scan and payload fetch returns one internally consistent snapshot. |
| Coalescing | Insert→patch, patch→patch, patch→delete, and insert→delete follow section 6.5. |
| Mutation limit | A 1,001-document change fails before commit; no partial state survives. |
| Whole-revision poll | Every row of a 1,000-document revision is dispatched before the cursor advances. |
| Empty change revision | Consumer advances to state watermark for catalog-only commits and write-then-cancel mutations with no document rows. |
| Pruning reset | Cursor older than the complete-revision watermark triggers a fresh subscription query. |
| Notification loss | Periodic polling catches up without NOTIFY. |
| Subscription startup race | Commit during initial query is retained and cannot be permanently missed. |
| Subscription rerun race | Invalidation during rerun causes another execution when required. |
| Subscription isolation | A change to an unrelated table does not dirty a single-table subscription. |
| Scalar ordering | Logical order equals Rust byte order and PostgreSQL BYTEA order. |
| Compound ranges | Every equality/range combination matches an independent logical reference implementation. |
| Encoding golden vectors | Persistent document/index fixtures remain stable for each format version. |
| Missing/null | Insert, decode, patch Keep/Set/Remove, index extraction, and uniqueness follow declared semantics. |
| Unique race | Two processes attempting the same unique key yield one commit and one structured conflict. |
| Unique missing/null | Multiple skipped unique entries are accepted; fully present duplicates are rejected. |
| Cursor directions | Duplicate logical keys paginate without loss in ascending and descending order. |
| Key maximum | Boundary and oversized keys behave correctly on every supported PostgreSQL build. |
| Payload evolution | Optional additions decode old maps; unsupported versions and duplicate fields fail. |
| Stale binary | Old compiled definitions cannot write after incompatible catalog activation. |
| Backfill block | Target-table mutations fail while Ready-index reads and other-table mutations continue. |
| Backfill crash | Checkpoint and table block survive; resume does not skip or duplicate logical coverage. |
| Backfill worker lock | A second worker cannot run the active job concurrently. |
| Backfill failure | Duplicate unique keys and oversized keys never expose an incomplete Ready index. |
| Unknown commit outcome | The runtime does not blindly replay a non-idempotent mutation. |
| Resource limits | Oversized documents, excessive nesting, query budgets, and timeouts fail predictably. |

An in-memory mutex is not evidence for the PostgreSQL writer-gate protocol.

# Appendix B. Repository layout and dependencies

## B.1 Workspace

Keep one public runtime crate and one proc-macro implementation crate:

```text
rustvex/
├── Cargo.toml
├── Cargo.lock
├── README.md
├── ARCHITECTURE.md
├── AUTH.md
├── crates/
│   ├── rustvex/
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   │   └── 0001_initial.sql
│   │   ├── examples/
│   │   │   └── notes.rs
│   │   ├── tests/
│   │   ├── benches/
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── db.rs
│   │       ├── error.rs
│   │       ├── id.rs
│   │       ├── value.rs
│   │       ├── schema/
│   │       ├── document/
│   │       ├── codec/
│   │       ├── index/
│   │       ├── query/
│   │       ├── transaction/
│   │       ├── revision/
│   │       ├── subscription/
│   │       ├── migration/
│   │       └── storage/
│   │           └── postgres.rs
│   └── rustvex-macros/
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           ├── table.rs
│           ├── values.rs
│           └── patch.rs
└── docs/
```

The macro crate is re-exported by `rustvex`. Users depend only on `rustvex`.
Do not split the runtime into more publishable crates in v1.

## B.2 Runtime dependencies

| Dependency | Purpose |
|---|---|
| `tokio` | Async runtime. |
| `tokio-postgres` | Direct PostgreSQL protocol; no ORM. |
| `deadpool-postgres` | Lightweight connection pool. |
| `bytes` | Efficient storage-boundary buffers. |
| `serde` | Internal serialization traits. |
| `rmp-serde` | Named-field MessagePack bodies. |
| `serde_json` | Logical `Json` values. |
| `uuid` | Initial UUIDv7-based 16-byte ID generation. |
| `blake3` | Schema and normalized-query fingerprints. |
| `base64` | Opaque cursor transport. |
| `thiserror` | Structured errors. |
| `tracing` | Diagnostic spans and structured logs. |
| `metrics` | Exporter-neutral counters and histograms. |
| `futures-core` | Subscription stream interface. |
| `rustvex-macros` | Re-exported schema/input macros. |

Use workspace dependency versions and minimal feature sets. Keep pool, driver,
MessagePack, UUID, and metrics implementation types out of the public core API.

Do not add Diesel, SQLx, SeaORM, Toasty, a general SQL builder, or a general
migration framework in v1.

## B.3 Macro and development dependencies

`rustvex-macros` uses `proc-macro2`, `quote`, and `syn`.

Development dependencies:

| Dependency | Purpose |
|---|---|
| `proptest` | Ordered encoding and query-range properties. |
| `testcontainers` | Real isolated PostgreSQL integration tests. |
| `trybuild` | Macro compile-pass and compile-fail tests. |
| `criterion` | Codec, planner, and hot-path benchmarks. |

# Appendix C. PostgreSQL and library references

These references support lower-level mechanics. Rustvex's higher-level
protocols remain design decisions.

- [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [PostgreSQL LISTEN](https://www.postgresql.org/docs/current/sql-listen.html)
- [PostgreSQL NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html)
- [PostgreSQL B-tree ordering](https://www.postgresql.org/docs/current/indexes-ordering.html)
- [PostgreSQL B-tree implementation](https://www.postgresql.org/docs/current/btree.html)
- [PostgreSQL binary data types](https://www.postgresql.org/docs/current/datatype-binary.html)
- [PostgreSQL routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [PostgreSQL advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)
- [rmp-serde `to_vec_named`](https://docs.rs/rmp-serde/latest/rmp_serde/encode/fn.to_vec_named.html)
- [tokio-postgres transaction](https://docs.rs/tokio-postgres/latest/tokio_postgres/struct.Transaction.html)

**Implementation status:** this specification does not establish that Rustvex
exists, passes the acceptance suite, or meets any unmeasured performance target.
