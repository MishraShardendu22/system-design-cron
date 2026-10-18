# DAY 7: Data Modeling and Access Patterns

**Progression level:** Level 1, Foundations and architectural reasoning  
**Concept ID:** SD-007  
**Prerequisites:** SD-003, Capacity estimation and workload modeling; SD-006, Domain boundaries and data ownership, covered  
**Status:** Lesson delivered; exercises pending.

### 1. Why It Exists

A data model is an executable statement of what the system considers true and how callers are allowed to find and change that truth. Poor models emerge when teams design tables from nouns in a product brief instead of from invariants, lifecycle transitions, and query patterns. The result is a schema that cannot enforce important constraints, forces expensive joins or scans on critical paths, duplicates facts inconsistently, or makes routine changes require broad rewrites.

A system can have correct API endpoints and clear domain ownership yet fail at scale if its most common query scans millions of rows, tenant boundaries are missing from keys, a mutable field is used as an unstable pagination cursor, or a uniqueness rule exists only in application code. Conversely, prematurely denormalizing every read can create update anomalies and complicated repair logic before measurements justify the cost.

Data modeling connects domain invariants to persistence structures and workload. The goal is not to normalize everything or optimize every query in advance. It is to preserve correctness while making known access paths efficient and leaving room for measured evolution.

### 2. Core Idea

Model data in this order:

1. **Facts and invariants:** identify canonical facts, valid states, uniqueness, ownership, and relationships.
2. **Lifecycle:** define creation, mutation, deletion, retention, and archival transitions.
3. **Access patterns:** enumerate reads and writes, filters, sort order, cardinality, frequency, and freshness requirements.
4. **Logical model:** choose entities, attributes, relationships, and normalization boundaries.
5. **Physical model:** choose primary keys, indexes, constraints, partitioning candidates, and storage representation.
6. **Validation:** use representative data and query plans to test whether the model preserves invariants and meets workload objectives.

A logical model describes meaning. A physical model describes how a selected database stores and retrieves it. An access pattern is a concrete operation, such as “list the most recently created active documents for one tenant.” “Support documents” is not an access pattern.

**Normalization** reduces duplicated authoritative facts and update anomalies. **Denormalization** duplicates or precomputes data to improve specific reads, at the cost of write amplification, freshness management, and repair. Neither is universally correct. Keep canonical facts normalized enough to preserve invariants, then introduce derived representations for measured read or aggregation needs.

Use database constraints for invariants the database can enforce: primary keys, foreign keys, NOT NULL, CHECK, and unique constraints. Application validation improves error messages but does not replace constraints under concurrent writers. An application-level “check, then insert” can race when two transactions perform the check simultaneously.

Keys are part of the model. A key should have defined uniqueness scope, mutability, distribution characteristics, and exposure rules. A surrogate immutable ID is often useful for identity; natural keys may still need unique constraints. In multi-tenant systems, tenant identity belongs in authorization predicates and often in composite indexes or uniqueness scopes, even when a globally unique document ID is used.

### 3. Mental Model

Treat each table and index as a materialized answer to a set of questions.

- A **row** represents a fact or entity state.
- A **constraint** defines which states are legal.
- An **index** is an auxiliary access structure that trades storage and write work for faster lookup or ordering.
- A **query** is a workload contract: predicates, ordering, projection, result limit, and consistency expectations.
- A **derived view** is a copy or precomputation whose freshness and repair policy must be explicit.

For every critical query, trace the path from its predicate and ordering to an index, then estimate how many rows and pages it must examine and return. An index is useful only if its column order and operator support match the query shape and selectivity. An index can also hurt writes because each mutation must maintain it.

This model has a limitation: indexes do not guarantee a fast query. The optimizer chooses plans using statistics and cost estimates; parameter distributions, stale statistics, table size, cache state, lock contention, and result volume affect latency. Validate with realistic data and the target database's EXPLAIN facilities.

### 4. Real-World Example

Consider a multi-tenant document service. Users create documents, fetch a document by ID, list recent documents in a tenant, and update titles. Metadata is relational; large binary content is stored separately.

**Documented facts:** PostgreSQL documents primary keys, foreign keys, unique and check constraints, indexes, and EXPLAIN. Its documentation explains that indexes can improve retrieval but impose overhead. References: [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html), [PostgreSQL indexes](https://www.postgresql.org/docs/current/indexes.html), and [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html).

**Established industry practice:** derive indexes from query predicates and ordering, constrain tenant-scoped reads, paginate large result sets, and validate plans against representative data. Keep large blobs outside relational rows when their size and lifecycle justify that separation.

**Engineering inference for this hypothetical service:** the hot list query filters by tenant, excludes deleted rows, orders by immutable creation sequence descending, and applies a limit. A composite partial index on tenant and creation sequence for non-deleted rows may serve that path. A globally unique document ID supports direct lookup, but the query must still include tenant scope so authorization is enforced in the data access path. These choices must be validated with actual tenant skew and query plans, not treated as universal rules.

### 5. Low-Level Design (LLD)

#### Logical entities and invariants

- **Tenant:** owns a namespace and policy boundary.
- **Document:** canonical metadata, tenant, immutable ID, title, creation sequence, timestamps, version, and optional deletion marker.
- **Document content:** binary object referenced by a content key, with independent retention and integrity policy.
- **Audit record:** append-only record of security-relevant mutations, with actor, tenant, entity, action, and timestamp.

Core invariants:

1. A document belongs to exactly one tenant.
2. A document ID is immutable and globally unique in this design.
3. A tenant may not read or mutate another tenant's document.
4. The creation sequence is immutable and unique per tenant.
5. A successful update increments version; stale optimistic updates fail rather than silently overwrite newer state.
6. A soft-deleted document is excluded from ordinary list and read paths unless the API explicitly supports recovery.
7. Content references are not proof of authorization. Access is checked through document metadata before content is served.

A PostgreSQL-oriented schema:

~~~sql
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE documents (
    document_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(tenant_id),
    created_seq BIGINT GENERATED ALWAYS AS IDENTITY,
    title TEXT NOT NULL CHECK (length(btrim(title)) BETWEEN 1 AND 200),
    content_key TEXT NOT NULL,
    version BIGINT NOT NULL DEFAULT 1 CHECK (version > 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ,
    UNIQUE (tenant_id, created_seq)
);

CREATE INDEX documents_active_tenant_created
    ON documents (tenant_id, created_seq DESC)
    WHERE deleted_at IS NULL;

CREATE INDEX documents_active_tenant_updated
    ON documents (tenant_id, updated_at DESC, document_id DESC)
    WHERE deleted_at IS NULL;

CREATE TABLE document_audit (
    audit_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id UUID NOT NULL,
    document_id UUID NOT NULL,
    actor_id UUID NOT NULL,
    action TEXT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    details JSONB NOT NULL DEFAULT '{}'::jsonb
);

CREATE INDEX document_audit_tenant_time
    ON document_audit (tenant_id, occurred_at DESC, audit_id DESC);
~~~

The identity sequence is not guaranteed to be gapless or commit-ordered. It is usable as a stable ordering key here, not as a count of rows or strict global commit timestamp. If the product requires a per-tenant commit-ordered sequence, that requirement needs a different allocation protocol and a corresponding contention tradeoff. A production migration should also consider whether the audit table needs foreign keys, partitioning, retention, or immutable storage guarantees.

#### Query contracts

| Operation | Predicate and order | Model decision |
|---|---|---|
| Get document | tenant ID and document ID | Scope reads to tenant and ID |
| List recent documents | tenant, active rows, creation sequence descending | Composite partial index and keyset cursor |
| List recently updated | tenant, active rows, updated time and ID descending | Separate index because ordering differs |
| Update title | tenant, document ID, expected version | Optimistic concurrency |
| Audit history | tenant, time and tie-breaker | Append-only records and time-oriented index |

A single index rarely serves all these patterns. Index order matters: a composite B-tree on tenant and creation sequence is not equivalent to creation sequence and tenant for tenant-filtered traversal. Indexes should be justified by observed query plans and write overhead.

For keyset pagination, return a cursor containing the last row's ordering key. The next page uses creation sequence less than the last sequence for descending order. A cursor should be opaque to clients and validated by the server. It does not automatically provide a snapshot across multiple requests; concurrent inserts may appear before the current cursor, while deletes and updates can change later results. Snapshot exports require a defined snapshot or watermark contract.

### 6. High-Level Design (HLD)

~~~mermaid
flowchart LR
    Client --> API[Document API]
    API --> Auth[Tenant and resource authorization]
    Auth --> Repo[Document repository]
    Repo --> DB[(Relational metadata store)]
    API --> Blob[Content access adapter]
    Blob --> Object[(Object storage)]
    DB --> Audit[Audit or outbox transaction]
    Audit --> Events[Durable event transport]
    Events --> Search[Search projection]
~~~

Read flow: authenticate the principal, derive the authorized tenant scope, query metadata with tenant predicates and bounded result size, then retrieve content only after the metadata authorization decision. The database index supports the requested access path. Search is a derived projection and must not bypass canonical authorization.

Write flow: validate the command, execute a conditional metadata update or insert, persist audit or outbox state in the same database transaction when required, commit, then publish the outbox asynchronously. If content upload and metadata commit use different storage systems, the workflow needs explicit states and cleanup or reconciliation for orphaned objects or missing references.

Scaling behavior: read replicas or caches may later serve eligible reads, but freshness requirements must be stated. Large tenants can dominate index pages and buffer cache; tenant skew must be measured. Partitioning should be considered only after workload and operational evidence show a need, because it changes uniqueness, query routing, migration, and maintenance constraints.

### 7. Realistic Go Code

The following repository implementation uses database/sql and PostgreSQL-style placeholders. It demonstrates tenant-scoped queries, keyset pagination, bounded page size, and optimistic concurrency. Supply a PostgreSQL driver when constructing sql.DB; no driver is imported here. The schema above is a prerequisite migration.

~~~go
package documents

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "strings"
    "time"
)

var (
    ErrNotFound = errors.New("document not found")
    ErrVersionConflict = errors.New("document version conflict")
    ErrInvalidInput = errors.New("invalid input")
)

type Document struct {
    ID         string
    TenantID   string
    Title      string
    ContentKey string
    CreatedSeq int64
    Version    int64
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

type Repository struct {
    db *sql.DB
}

func NewRepository(db *sql.DB) (*Repository, error) {
    if db == nil {
        return nil, errors.New("database is required")
    }
    return &Repository{db: db}, nil
}

func (r *Repository) Get(ctx context.Context, tenantID, documentID string) (Document, error) {
    if tenantID == "" || documentID == "" {
        return Document{}, ErrInvalidInput
    }

    const q = `
        SELECT document_id::text, tenant_id::text, title, content_key,
               created_seq, version, created_at, updated_at
        FROM documents
        WHERE tenant_id = $1::uuid
          AND document_id = $2::uuid
          AND deleted_at IS NULL`

    var d Document
    err := r.db.QueryRowContext(ctx, q, tenantID, documentID).Scan(
        &d.ID, &d.TenantID, &d.Title, &d.ContentKey,
        &d.CreatedSeq, &d.Version, &d.CreatedAt, &d.UpdatedAt,
    )
    if errors.Is(err, sql.ErrNoRows) {
        return Document{}, ErrNotFound
    }
    if err != nil {
        return Document{}, fmt.Errorf("get document: %w", err)
    }
    return d, nil
}

type Page struct {
    Items      []Document
    NextCursor int64
    HasMore    bool
}

func (r *Repository) ListRecent(
    ctx context.Context,
    tenantID string,
    beforeSeq int64,
    limit int,
) (Page, error) {
    if tenantID == "" || limit < 1 || limit > 100 || beforeSeq < 0 {
        return Page{}, ErrInvalidInput
    }

    const q = `
        SELECT document_id::text, tenant_id::text, title, content_key,
               created_seq, version, created_at, updated_at
        FROM documents
        WHERE tenant_id = $1::uuid
          AND deleted_at IS NULL
          AND ($2 = 0 OR created_seq < $2)
        ORDER BY created_seq DESC
        LIMIT $3`

    rows, err := r.db.QueryContext(ctx, q, tenantID, beforeSeq, limit+1)
    if err != nil {
        return Page{}, fmt.Errorf("list documents: %w", err)
    }
    defer rows.Close()

    items := make([]Document, 0, limit+1)
    for rows.Next() {
        var d Document
        if err := rows.Scan(
            &d.ID, &d.TenantID, &d.Title, &d.ContentKey,
            &d.CreatedSeq, &d.Version, &d.CreatedAt, &d.UpdatedAt,
        ); err != nil {
            return Page{}, fmt.Errorf("scan document: %w", err)
        }
        items = append(items, d)
    }
    if err := rows.Err(); err != nil {
        return Page{}, fmt.Errorf("iterate documents: %w", err)
    }

    page := Page{Items: items}
    if len(items) > limit {
        page.HasMore = true
        page.Items = items[:limit]
        page.NextCursor = page.Items[len(page.Items)-1].CreatedSeq
    } else if len(items) > 0 {
        page.NextCursor = items[len(items)-1].CreatedSeq
    }
    return page, nil
}

func (r *Repository) UpdateTitle(
    ctx context.Context,
    tenantID, documentID string,
    expectedVersion int64,
    title string,
) (Document, error) {
    title = strings.TrimSpace(title)
    if tenantID == "" || documentID == "" || expectedVersion < 1 ||
        title == "" || len([]rune(title)) > 200 {
        return Document{}, ErrInvalidInput
    }

    const q = `
        UPDATE documents
        SET title = $4, version = version + 1, updated_at = now()
        WHERE tenant_id = $1::uuid
          AND document_id = $2::uuid
          AND version = $3
          AND deleted_at IS NULL
        RETURNING document_id::text, tenant_id::text, title, content_key,
                  created_seq, version, created_at, updated_at`

    var d Document
    err := r.db.QueryRowContext(
        ctx, q, tenantID, documentID, expectedVersion, title,
    ).Scan(
        &d.ID, &d.TenantID, &d.Title, &d.ContentKey,
        &d.CreatedSeq, &d.Version, &d.CreatedAt, &d.UpdatedAt,
    )
    if errors.Is(err, sql.ErrNoRows) {
        return Document{}, ErrVersionConflict
    }
    if err != nil {
        return Document{}, fmt.Errorf("update document title: %w", err)
    }
    return d, nil
}
~~~

The implementation intentionally maps a zero-row conditional update to a generic conflict so it does not reveal whether a cross-tenant ID exists. Production code should validate UUID syntax at the boundary, map SQLSTATE errors deliberately, configure pool limits and connection lifetime, and instrument query duration. A zero cursor is used as a sentinel and assumes valid sequences are positive; a production cursor should use an explicit optional value. Cursor encoding, integrity protection, expiry, and filter binding belong in the API layer. A single SQL query does not provide a stable snapshot across independent page requests.

### 8. Example Usage

A caller creates the repository after opening a database connection with a PostgreSQL driver:

~~~go
repo, err := NewRepository(db)
if err != nil {
    return err
}

ctx, cancel := context.WithTimeout(parent, 250*time.Millisecond)
defer cancel()

page, err := repo.ListRecent(ctx, tenantID, 0, 50)
if err != nil {
    return fmt.Errorf("list recent documents: %w", err)
}
for _, doc := range page.Items {
    fmt.Printf("%s %s v%d\\n", doc.ID, doc.Title, doc.Version)
}
if page.HasMore {
    fmt.Printf("next cursor: %d\\n", page.NextCursor)
}
~~~

Expected behavior: the first call returns at most 50 active documents for the authorized tenant, newest creation sequence first. HasMore is true only when the query returned the extra row. The next request passes NextCursor as beforeSeq; the query continues strictly below that sequence. An empty result has no next page. Invalid limits are rejected before database work.

A stale title update returns ErrVersionConflict, preventing a lost update. A missing document read returns ErrNotFound. Query cancellation or database timeout is surfaced as an error and must not be interpreted as proof that a write did not commit.

For integration tests, run against a disposable PostgreSQL instance and verify:
- tenant A cannot read or update tenant B's document even with a known ID;
- list results are ordered and page boundaries do not duplicate immutable sequence values;
- deleted rows are excluded;
- stale versions cannot overwrite newer titles;
- concurrent inserts and deletes have the documented non-snapshot pagination behavior;
- query plans use the intended index for representative tenant sizes.

### 9. Production Considerations

- **Query workload:** collect normalized query fingerprints, frequency, latency percentiles, rows returned, and plan changes. Avoid logging raw sensitive parameters.
- **Index cost:** every extra index consumes disk and cache and increases insert/update/delete work. Remove unused indexes only after checking rare query and operational requirements.
- **Tenant isolation:** derive tenant identity from authenticated context, not an untrusted query parameter. Apply tenant predicates to reads, updates, deletes, and joins. Database row-level security can provide defense in depth, not replace correct authorization.
- **Cardinality and skew:** test small and very large tenants. A design that performs well for median tenant size may degrade for a dominant tenant.
- **Pagination:** bound page size, use deterministic ordering with a unique tie-breaker, validate cursor scope, and document whether pages represent a live view or snapshot.
- **Concurrency:** use constraints and conditional updates for invariants. Transactions and optimistic versions address different concerns; choose isolation and retry behavior deliberately.
- **Consistency:** define when a successful write becomes visible to list, search, cache, and replica reads. Derived projections need lag metrics and rebuild paths.
- **Blob lifecycle:** metadata and object storage do not share an ordinary relational transaction. Model pending/ready/deleted states, cleanup, checksums, and orphan reconciliation.
- **Migrations:** use expand-and-contract changes, backfill in bounded batches, dual-read or dual-write only with explicit reconciliation, and verify before removing old fields or indexes.
- **Pool management:** cap open and idle connections relative to database capacity, set acquisition and query deadlines, and monitor pool wait time.
- **Security and privacy:** classify fields, encrypt sensitive values where appropriate, minimize retention, and ensure audit records do not copy secrets.
- **Observability:** measure query latency, rows scanned versus returned where available, lock waits, deadlocks, pool saturation, replication lag, index size, and migration progress.
- **Recovery:** document backup/restore, schema compatibility, projection rebuild, and data repair procedures.
- **Cost:** balance storage and index overhead against query latency and compute. Validate with representative data volume before committing to denormalized copies.

### 10. Tradeoffs

| Choice | Benefits | Costs and risks | Appropriate when |
|---|---|---|---|
| Normalized relational schema | Strong constraints and fewer update anomalies | Joins and read-time computation | Canonical transactional facts and related updates |
| Denormalized read projection | Fast specialized reads | Staleness, duplicate storage, repair logic | Measured read bottleneck or complex read shape |
| Composite B-tree index | Efficient equality filters plus ordered traversal | Write amplification and storage | Stable predicates/order match its leading columns |
| Offset pagination | Simple page numbers and random access | Deep offsets scan/discard rows; writes shift positions | Small, relatively static result sets |
| Keyset pagination | Efficient continuation and stable immutable ordering | No arbitrary page jump; cursor complexity | Large changing collections |
| Optimistic version column | Detects stale updates without long locks | Caller must resolve conflicts | Concurrent edits where conflicts are uncommon |
| Pessimistic row locking | Serializes conflicting mutations | Lock contention and deadlock risk | Short critical transactions requiring exclusive coordination |
| Soft deletion | Recovery and auditability | Every relevant query must filter deleted rows | Restore or retention requirements |
| Hard deletion | Simpler active data model and storage reclamation | Recovery/audit constraints | Policy permits irreversible deletion |

The appropriate model follows the invariant and workload. Do not add a search engine, cache, partitioning, or denormalized table solely because the system may grow. Establish the canonical model first, measure the actual bottleneck, then add the smallest structure that addresses it.

### 11. Connection to Previous Lessons

Day 3, SD-003, supplies request rates, data growth, cardinality, and peak workload estimates. Day 6, SD-006, defines canonical ownership and the invariants that the schema must preserve. Day 5, SD-005, supplies API semantics such as pagination, optimistic conflict responses, and authorization-visible behavior. Day 4, SD-004, provides latency objectives against which query plans are evaluated.

The prerequisites declared by the curriculum, SD-003 and SD-006, are covered. Earlier exercises remain pending because no learner submissions have been reviewed; they do not block this concept.

### 12. What Comes Next

**SD-008: Relational Databases and Transaction Semantics**, Level 2. The model now identifies entities, constraints, indexes, and query paths. The next step is to understand how transactions, isolation levels, locking, and commit behavior preserve those invariants when concurrent operations race or failures occur. SD-011 (NoSQL models), SD-012 (object storage), SD-013 (caching), and SD-015 (proxies) are also eligible based on the current dependency graph, but SD-008 is next in the storage progression. It is not taught here.

### 13. Mini Exercise

For an audit-event API, support a tenant-scoped endpoint that returns events newest first using a cursor and limit.

**Requirements:** define the event row, tenant isolation invariant, deterministic ordering, cursor fields, index, maximum page size, and behavior when new events arrive between requests. Include the SQL query and explain why the chosen index matches its predicates and ordering.

**Acceptance criteria:** no cross-tenant rows are returned; ordering has a unique tie-breaker; page size is bounded; continuation does not use a growing offset; cursor snapshot guarantees are explicit; and the index tradeoff is discussed. Do not provide the complete solution in this lesson.

### Key Takeaways

- Start with invariants and access patterns, not just entity names.
- Distinguish logical meaning from physical storage and indexing.
- Enforce concurrency-sensitive invariants with database constraints and conditional writes.
- Index predicates and ordering that matter to measured workload; indexes also cost writes and storage.
- Tenant identity must be part of authorization and the data-access path.
- Keyset pagination improves continuation for large collections but is not automatically snapshot-consistent.
- Derived data needs freshness, rebuild, and repair contracts.
- Use realistic query plans and production-like distributions to validate a model.

### Concept Map

~~~mermaid
flowchart TD
    A[SD-003: Capacity and workload] --> C[SD-007: Data modeling and access patterns]
    B[SD-006: Domain boundaries and ownership] --> C
    D[SD-005: API contracts and pagination] -. informs .-> C
    C --> E[SD-008: Relational transactions]
    C --> F[SD-011: NoSQL model selection]
    C --> G[SD-012: Object storage]
    C --> H[SD-013: Caching]
    E --> I[SD-009: Index structures and query planning]
    E --> J[SD-010: Query optimization and pooling]
~~~

### Exercise

Design the persistence layer for a multi-tenant document platform expected to grow from 10 million to 500 million metadata records. Workload estimates: 70% metadata reads, 20% list queries, 8% updates, and 2% creates/deletes; tenant sizes are highly skewed; list requests require p95 under 80 ms at the service boundary. Treat these numbers as exercise inputs, not facts about a real service.

**Deliverables:**
1. A logical entity/relationship model and a PostgreSQL-oriented schema with constraints.
2. An access-pattern inventory covering direct reads, tenant listing, recently updated documents, title updates, audit history, and deletion.
3. Index definitions with an explanation of leading-column order, partial predicates, and write costs.
4. A keyset-pagination contract, including ordering, cursor validation, concurrent-write behavior, and snapshot limitations.
5. Go repository interfaces and SQL for tenant-scoped reads, list pagination, and optimistic updates.
6. A load-test plan with realistic tenant skew, dataset sizes, concurrency, and latency measurements.
7. A migration plan for adding an index and backfilling a new field without blocking normal traffic.
8. Failure analysis for stale updates, query timeout, pool exhaustion, partial blob/metadata operations, and projection lag.

**Acceptance criteria:** constraints enforce the stated invariants; tenant scope is applied to all operations; query shapes have supporting indexes justified by plans; pagination has deterministic continuation semantics; stale writes are detected; the plan covers representative skew and load; migration has rollback and reconciliation; and no benchmark claim is made without measured results. This exercise remains pending until submitted and reviewed.

### Progress

- **Day:** 7, based on verified lesson history through Day 6 and absence of a Day 7 lesson on main.
- **Progression level:** Level 1, Foundations and architectural reasoning.
- **Current concept:** SD-007, Data modeling and access patterns.
- **Lesson path:** system-design/lessons/day-07-data-modeling-and-access-patterns.md
- **Prerequisites satisfied:** SD-003 and SD-006 are covered.
- **Mini exercise status:** Pending learner submission.
- **Substantial exercise status:** Pending learner submission.
- **Concepts covered after this lesson:** SD-001 through SD-007.
- **Next eligible concept selected:** SD-008, Relational databases and transaction semantics. SD-011, SD-012, SD-013, and SD-015 are also eligible by prerequisites, but SD-008 is next in the storage progression.
- **Repository update status:** Lesson delivery prepared from verified repository state. Final persistence and read-back verification must be recorded only after the lesson, curriculum, and progress updates are committed.

### Most Important Mental Model

**A good data model encodes domain invariants and makes the system's real access patterns efficient, with every index and derived copy paying for itself in measured workload.**
