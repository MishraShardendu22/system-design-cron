# DAY 11: NoSQL Data Models and Selection

**Progression level:** Level 2, Storage and request-path mechanics  
**Concept ID:** SD-011  
**Prerequisites:** SD-007, Data modeling and access patterns; SD-003, Capacity estimation and workload modeling, covered.  
**Status:** Lesson delivered; exercises pending.

### 1. Why It Exists

Choosing a database by category labels or benchmark headlines is a poor architecture decision. “NoSQL” covers storage models with materially different query, partitioning, transaction, indexing, and operational behavior. A key-value store optimized for direct key access is not automatically a good fit for arbitrary filtering. A document database can represent an aggregate naturally but still perform poorly when a critical query has no supporting index. A wide-column store can sustain distributed workloads when data is modeled around its partition and clustering keys, but that model constrains query flexibility. A graph database can make relationship traversals natural while adding a different operational and modeling cost.

The choice affects more than read latency. It determines how invariants are enforced, how data is partitioned, what queries are affordable, how writes are amplified, how hot keys behave, how data is repaired, and how the system evolves when access patterns change. Replacing a database later can require backfills, dual writes, reconciliation, API changes, and a migration period with two failure surfaces.

The goal is not to prove that one model is universally best. It is to select a model that satisfies correctness and operational constraints for the known workload, identify where it will stop fitting, and validate the decision with representative measurements.

### 2. Core Idea

Start from concrete access patterns and correctness requirements, then select a storage model. Do not start from a vendor or a desire to use a particular technology.

For every critical operation, record:

1. **Predicate:** which keys or attributes identify the records?
2. **Ordering and pagination:** is ordering required, and what is the stable continuation key?
3. **Cardinality:** how many records may match, per tenant and at peak?
4. **Read/write mix:** how often is the operation executed, and what does each mutation maintain?
5. **Consistency:** must a read observe a just-acknowledged write, or is bounded staleness acceptable?
6. **Atomicity:** which fields or entities must change together?
7. **Growth and skew:** can one tenant, key, partition, or relationship dominate traffic?
8. **Operational constraints:** backup, restore, repair, regional behavior, tooling, staffing, and cost.

Then compare candidate models against hard constraints first. A candidate that cannot preserve a required invariant or serve a critical access path is not rescued by a high aggregate score elsewhere. For the remaining candidates, use a weighted score only to organize discussion, not to manufacture precision. Benchmark the finalists with representative data, skew, concurrency, failure behavior, and operational tasks.

**Key-value:** retrieve or mutate a value using a known key. Secondary query capabilities vary by product.  
**Document:** store a structured aggregate, often with nested attributes and indexes over selected fields.  
**Wide-column:** organize rows by partition key and clustering order, typically designing tables around known queries.  
**Graph:** represent entities as vertices and relationships as edges, making traversals across relationships a primary operation.

These are logical model families, not interchangeable implementation guarantees. Products within a family differ in indexing, transaction scope, consistency options, limits, and failure semantics. See the official [DynamoDB data-modeling guidance](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/data-modeling.html), [MongoDB data modeling documentation](https://www.mongodb.com/docs/manual/data-modeling/), [Cassandra data-modeling introduction](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/intro.html), and [Neo4j data-modeling guide](https://neo4j.com/docs/getting-started/data-modeling/).

### 3. Mental Model

A database model is a set of query and invariant tradeoffs.

- A **key-value model** makes identity and direct access explicit. If callers need an unbounded set of predicates, those predicates require supported indexes, a separate query path, or a different model.
- A **document model** groups fields that are commonly read or updated together. Embedding can make an aggregate read efficient, but unbounded arrays, duplicated mutable facts, and large documents create growth and update problems.
- A **wide-column model** places rows into partitions and orders rows within each partition. Partition design controls both query locality and the distribution of load and data. A partition that grows without bound or receives disproportionate traffic is a design defect, not merely a tuning issue.
- A **graph model** makes relationships first-class. It is valuable when traversals are central to the workload, but the graph still needs bounded traversal patterns, selective starting points, and controls for high-degree nodes.

NoSQL does not mean “no schema,” “no transactions,” or “eventual consistency by definition.” Every system has a data contract, even when the database does not enforce a relational schema. Consistency and transaction guarantees are product- and configuration-specific. For example, DynamoDB documents different read-consistency options, while Cassandra exposes consistency levels and replica behavior. Consult the [DynamoDB read-consistency documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html) and the [Cassandra architecture documentation](https://cassandra.apache.org/doc/latest/cassandra/architecture/).

The central question is: **Which data model makes the critical access paths and correctness rules explicit while keeping partitioning, write amplification, and operations within budget?**

### 4. Real-World Example

Consider a multi-tenant device registry for a cloud platform. Each tenant owns devices with mutable status, timestamps, firmware metadata, and flexible vendor attributes. The API supports:

- Fetch one device by tenant and device ID.
- List devices for a tenant and status, newest update first.
- Update status only if the caller's observed version is current.
- Find devices by a small, explicitly supported set of indexed attributes.
- Occasionally traverse dependency relationships between devices.

**Documented facts:** DynamoDB documents partition-key design and data modeling; MongoDB documents embedding and references; Cassandra describes query-driven table modeling; Neo4j documents nodes, relationships, and graph modeling. References: [DynamoDB partition-key design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html), [MongoDB embedding versus references](https://www.mongodb.com/docs/manual/data-modeling/concepts/embedding-vs-references/), [Cassandra data modeling](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/intro.html), and [Neo4j data modeling](https://neo4j.com/docs/getting-started/data-modeling/).

**Established industry practice:** enumerate access patterns, quantify cardinality and skew, define consistency and conflict semantics, and benchmark the complete workload rather than a single point lookup. Treat indexes, replicas, derived views, and duplicated records as write and operational costs.

**Engineering inference for this hypothetical service:** a document store is a reasonable initial candidate if device metadata is aggregate-shaped, the status listing has a supported compound index, and conditional version updates meet the required concurrency semantics. A key-value model may be better if direct lookup dominates and the list paths are separately materialized. A wide-column model becomes more attractive for very high sustained write volume and predictable partitioned queries. A graph model is justified only if dependency traversals are frequent or complex enough to outweigh the cost of maintaining a graph representation. The workload numbers and decision must be validated; these are not universal product recommendations.

### 5. Low-Level Design (LLD)

#### Access-pattern contract

| Operation | Required key or predicate | Ordering and bound | Correctness requirement |
|---|---|---|---|
| Get device | tenant ID + device ID | One record | Tenant isolation |
| List by status | tenant ID + status | updated_at descending, device ID tie-breaker, bounded page | Stable ordering for a fixed snapshot; document behavior when records change between pages |
| Change status | tenant ID + device ID | One conditional mutation | Compare-and-swap using expected version |
| Filter by supported attribute | Explicit indexed attribute and tenant | Bounded result | No accidental collection-wide scan on a hot path |
| Traverse dependencies | Starting device ID + bounded edge types/depth | Explicit depth and result cap | Authorization and cycle handling |

The contract deliberately excludes arbitrary filters and unbounded graph traversal. Adding an API query is a storage-model change when it requires a new index, materialized view, table, or traversal strategy.

#### Candidate physical models

**Document collection example**

~~~javascript
{
  tenantId: "tenant-42",
  deviceId: "device-913",
  status: "active",
  updatedAt: ISODate("2026-10-23T07:00:00Z"),
  version: 17,
  attributes: {
    vendor: "acme",
    firmware: "4.2.1"
  }
}
~~~

A compound index could begin with tenantId and status, followed by updatedAt and deviceId in the required order. The adapter must use an atomic conditional update on version, not a separate read followed by an unconditional write. Verify the actual index and update semantics against the selected product.

**Key-value example**

~~~text
PK = TENANT#tenant-42
SK = DEVICE#device-913
status = active
updatedAt = 2026-10-23T07:00:00Z
version = 17

GSI1PK = TENANT#tenant-42#STATUS#active
GSI1SK = UPDATED#2026-10-23T07:00:00Z#DEVICE#device-913
~~~

The secondary index supports the tenant-and-status listing path, but index propagation and consistency guarantees must be understood. The partition-key design must also avoid a small set of overloaded keys. The exact syntax is illustrative, not a portable schema.

**Wide-column example**

~~~sql
CREATE TABLE devices_by_tenant_status (
    tenant_id text,
    status text,
    updated_at timestamp,
    device_id text,
    version bigint,
    attributes map<text, text>,
    PRIMARY KEY ((tenant_id, status), updated_at, device_id)
) WITH CLUSTERING ORDER BY (updated_at DESC, device_id ASC);
~~~

This table is organized for one query shape, not for arbitrary predicates. Validate the exact clustering order, supported mutation semantics, partition size, and query restrictions against the chosen Cassandra version. A separate table may be required for another access path, which means duplicated data and a defined update/reconciliation mechanism.

**Graph example**

~~~text
(:Tenant)-[:OWNS]->(:Device)
(:Device)-[:DEPENDS_ON]->(:Device)
(:Device)-[:REPORTS_TO]->(:Device)
~~~

Graph edges make dependency traversal explicit. Device identity and tenant authorization still require a defined lookup path and constraints. High-degree nodes, cycles, traversal depth, and tenant boundaries must be included in query design.

These examples are alternative physical designs, not four schemas to deploy simultaneously by default.

### 6. High-Level Design (HLD)

~~~mermaid
flowchart TD
  Client --> API[Authenticated API]
  API --> Contract[Validate access pattern and bounds]
  Contract --> Service[Domain service]
  Service --> Repo[Repository contract]
  Repo --> KV[(Key-value adapter)]
  Repo --> Doc[(Document adapter)]
  Repo --> Wide[(Wide-column adapter)]
  Repo --> Graph[(Graph adapter)]
  Service --> Telemetry[Latency, conflicts, errors, cardinality]
  Repo --> Telemetry
  Telemetry --> Decision[Benchmark and model review]
~~~

The diagram shows candidate adapters behind an explicit domain contract, not a recommendation to route each request to four databases. A production deployment normally selects one authoritative model for each bounded responsibility. A second store is justified when a separate access pattern warrants a derived representation and the system defines how that representation is populated, reconciled, and recovered.

Keep tenant authorization and domain invariants above the adapter boundary. The adapter must implement the declared atomicity and consistency contract; it must not silently weaken it because a backend lacks the required primitive. If no candidate can meet the contract, change the design or the requirement explicitly rather than hiding the gap.

### 7. Realistic Go Code

The service exposes only the access patterns the workload requires. A backend-specific adapter implements conditional updates and indexed listing using its native primitives. The interface is intentionally not a generic query builder.

~~~go
package devicecatalog

import (
    "context"
    "errors"
    "fmt"
    "strings"
    "time"
)

var (
    ErrInvalidInput = errors.New("invalid device request")
    ErrConflict = errors.New("device version conflict")
    ErrNotFound = errors.New("device not found")
    ErrUnsupportedQuery = errors.New("unsupported access pattern")
    ErrInvalidRepositoryResult = errors.New("invalid repository result")
)

type Device struct {
    TenantID string
    ID string
    Status string
    UpdatedAt time.Time
    Version int64
    Attributes map[string]string
}

type Cursor struct {
    UpdatedAt time.Time
    DeviceID string
}

type Repository interface {
    UpdateStatusIfVersion(
        ctx context.Context,
        tenantID, deviceID string,
        expectedVersion int64,
        status string,
        updatedAt time.Time,
    ) (Device, error)

    ListByTenantStatus(
        ctx context.Context,
        tenantID, status string,
        cursor *Cursor,
        limit int,
    ) ([]Device, *Cursor, error)
}

type Service struct {
    repo Repository
    now func() time.Time
}

func NewService(repo Repository, now func() time.Time) (*Service, error) {
    if repo == nil {
        return nil, errors.New("repository is required")
    }
    if now == nil {
        now = time.Now
    }
    return &Service{repo: repo, now: now}, nil
}

func validStatus(status string) bool {
    switch status {
    case "active", "inactive", "retired", "quarantined":
        return true
    default:
        return false
    }
}

func (s *Service) UpdateStatus(
    ctx context.Context,
    tenantID, deviceID string,
    expectedVersion int64,
    status string,
) (Device, error) {
    if ctx == nil ||
        strings.TrimSpace(tenantID) == "" ||
        strings.TrimSpace(deviceID) == "" ||
        expectedVersion < 1 ||
        !validStatus(status) {
        return Device{}, ErrInvalidInput
    }

    device, err := s.repo.UpdateStatusIfVersion(
        ctx, tenantID, deviceID, expectedVersion, status, s.now().UTC(),
    )
    if err != nil {
        return Device{}, fmt.Errorf("update device status: %w", err)
    }
    if device.TenantID != tenantID ||
        device.ID != deviceID ||
        device.Status != status ||
        device.Version != expectedVersion+1 {
        return Device{}, ErrInvalidRepositoryResult
    }
    return device, nil
}

func (s *Service) ListByStatus(
    ctx context.Context,
    tenantID, status string,
    cursor *Cursor,
    limit int,
) ([]Device, *Cursor, error) {
    if ctx == nil ||
        strings.TrimSpace(tenantID) == "" ||
        !validStatus(status) ||
        limit < 1 || limit > 100 ||
        (cursor != nil &&
            (cursor.UpdatedAt.IsZero() || strings.TrimSpace(cursor.DeviceID) == "")) {
        return nil, nil, ErrInvalidInput
    }

    devices, next, err := s.repo.ListByTenantStatus(
        ctx, tenantID, status, cursor, limit,
    )
    if err != nil {
        return nil, nil, fmt.Errorf("list devices by status: %w", err)
    }
    if len(devices) > limit {
        return nil, nil, ErrInvalidRepositoryResult
    }
    for _, device := range devices {
        if device.TenantID != tenantID || device.Status != status {
            return nil, nil, ErrInvalidRepositoryResult
        }
    }
    return devices, next, nil
}
~~~

The repository adapter must implement the version predicate and mutation atomically. A failed conditional write maps to ErrConflict; absence maps to ErrNotFound. Do not implement compare-and-swap as a read followed by an unconditional write. The adapter must enforce tenant-scoped predicates, the requested sort order, the limit, and a deterministic tie-breaker. It should return only supported query shapes and propagate context cancellation. Backend-specific errors must be mapped without discarding useful causes.

This is a service boundary, not a complete database driver. A production adapter still needs connection lifecycle management, serialization rules, index or table definitions, retry classification, metrics, and integration tests against the selected database.

### 8. Example Usage

~~~go
ctx, cancel := context.WithTimeout(request.Context(), 200*time.Millisecond)
defer cancel()

service, err := devicecatalog.NewService(repository, time.Now)
if err != nil {
    return err
}

updated, err := service.UpdateStatus(
    ctx, authenticatedTenantID, deviceID, expectedVersion, "quarantined",
)
if err != nil {
    return err
}
_ = updated

devices, nextCursor, err := service.ListByStatus(
    ctx, authenticatedTenantID, "active", decodedCursor, 50,
)
if err != nil {
    return err
}
_, _ = devices, nextCursor
~~~

The repository is constructed once from the chosen backend adapter. The authenticated tenant ID comes from trusted authorization context, not an unchecked client field. The API should encode the continuation cursor as an opaque, integrity-protected token and validate its tenant, status, and query version before passing it to the service. Cursor semantics must be documented for records that change status or updatedAt between page requests.

### 9. Production Considerations

- Benchmark representative cardinality, tenant skew, hot keys, payload sizes, read/write ratios, and peak concurrency. Uniform synthetic keys can conceal partition hotspots.
- Define the source of truth and the consistency contract for every duplicated or derived representation.
- Model partition growth and worst-case tenant size, not only average size. Set explicit limits for arrays, documents, rows per partition, graph traversal depth, and result count where the model requires them.
- Quantify secondary-index maintenance, replication, compaction, storage amplification, and backup footprint.
- Validate conditional updates, uniqueness, and multi-record invariants under concurrent writers. Application-level check-then-write logic is not a substitute for an atomic backend primitive.
- Distinguish acknowledged-write durability, read-your-writes behavior, stale secondary-index results, and cross-region consistency. Do not infer guarantees from the NoSQL label.
- Treat pagination as a consistency contract. A deterministic sort key avoids ambiguous ordering, but does not by itself create a snapshot across multiple requests.
- Track latency percentiles, throughput, error and conflict rates, throttling, retries, partition or shard skew, storage growth, and repair or compaction backlog.
- Retry only errors that are safe to retry. A timeout can leave the caller uncertain whether a write committed; use idempotency keys or conditional versions where required.
- Plan backup restore, point-in-time recovery where supported, schema or index changes, data export, and a migration rehearsal before committing to a model.
- If using multiple stores, define replayable change capture or an outbox-driven propagation path, lag objectives, reconciliation, tombstone handling, and repair ownership. Two independent writes do not become atomic because they run in one request handler.
- Review data lifecycle, retention, deletion, encryption, tenant isolation, and access-control semantics for every copy of the data.
- Re-run the decision when access patterns change materially. A new query can require a new index, collection, table, materialized view, or a different model.

### 10. Tradeoffs

| Model | Strong fit | Main costs and constraints | Warning signs |
|---|---|---|---|
| Key-value | Known-key lookup, simple mutation, low-latency access | Query flexibility depends on product; secondary indexes and derived keys need maintenance | Frequent scans or many ad hoc predicates |
| Document | Aggregate-shaped records, flexible fields, indexed document queries | Document growth, embedding/reference decisions, index and update costs | Unbounded nested collections or cross-document invariants |
| Wide-column | High-volume writes and predictable partitioned queries | Query-driven schema, denormalization, partition and clustering constraints | Hot or unbounded partitions, growing need for ad hoc queries |
| Graph | Relationship-heavy queries and multi-hop traversal | Traversal planning, graph operations, high-degree nodes, separate operational skills | Mostly simple key lookups with few relationship queries |
| Relational baseline | Rich joins, constraints, transactions, evolving query shapes | Horizontal distribution may require additional design and operations | Actual measured limits cannot meet required scale or availability |

Use the relational model as a baseline, not as a straw man. A relational database may remain the best choice when the workload depends on joins, constraints, and transactional invariants. Conversely, a NoSQL model is not justified merely by a large record count. The decision should be based on measured workload and required guarantees.

### 11. Connection to Previous Lessons

Day 3, SD-003, supplies workload rate, growth, payload, and skew estimates. Day 7, SD-007, establishes invariants and query-driven data modeling. Day 8, SD-008, establishes transaction and concurrency semantics; Day 9, SD-009, and Day 10, SD-010, explain index plans and query/pool bottlenecks. SD-011 extends these ideas to models where partition keys, denormalization, aggregate boundaries, or graph edges shape the available queries.

The previous exercises remain pending because no learner submissions have been reviewed. Lesson delivery is not evidence of mastery.

### 12. What Comes Next

**SD-012: Object Storage and Blob Lifecycle**, Level 2. It covers large-object placement, metadata separation, retention, integrity, lifecycle transitions, and transfer paths. Its prerequisite, SD-007, is covered. SD-013, SD-015, and SD-019 are also eligible after this lesson, but SD-012 is next in curriculum order. It is not taught here.

### 13. Mini Exercise

A device registry has 30 million records. Ninety percent of requests fetch a device by tenant and ID, eight percent list devices by tenant and status ordered by update time, and two percent traverse dependency relationships up to four hops. Writes update status and version. One tenant has 18% of all records and receives 35% of peak traffic. The product team proposes a single database chosen because a benchmark reports one million writes per second.

Choose a primary model and explain what evidence would change your decision. Specify the key/index or partition design for the two most frequent access patterns, the atomicity required for version updates, and how you would bound relationship traversal. Identify the risks introduced by the dominant tenant and describe a representative benchmark and failure test.

**Acceptance criteria:** State assumptions; compare at least three candidate models; account for access patterns, skew, correctness, index or denormalization cost, and operational recovery; and identify measurable selection criteria. Do not treat the benchmark headline as sufficient evidence.

### Key Takeaways

- Choose a model from explicit access patterns and invariants, not category labels or benchmark headlines.
- Key-value, document, wide-column, and graph models have different query and partitioning contracts.
- NoSQL does not imply schema-free operation or eventual consistency; guarantees are product- and configuration-specific.
- Partition skew, hot keys, unbounded collections, and high-degree graph nodes must be modeled explicitly.
- Conditional writes and uniqueness must be enforced atomically where required.
- Denormalization and secondary indexes trade read efficiency for write, storage, and repair cost.
- Multiple stores require a source of truth, propagation semantics, reconciliation, and recovery ownership.
- Validate with representative workload, skew, failure, and operational tests.

### Concept Map

~~~mermaid
flowchart TD
  A[SD-003 Workload and skew] --> C[SD-011 NoSQL model selection]
  B[SD-007 Access patterns and invariants] --> C
  D[Key-value: direct key access] --> C
  E[Document: aggregate and indexed fields] --> C
  F[Wide-column: partitioned query paths] --> C
  G[Graph: relationship traversal] --> C
  C --> H[Partition and index design]
  C --> I[Consistency and conditional writes]
  C --> J[Benchmark, recovery and operations]
  C --> K[SD-012 Object storage lifecycle]
  C --> L[SD-019 Replication fundamentals]
~~~

### Exercise

Design a multi-tenant device registry and submit a database-model decision record.

**Deliverables:** Workload and access-pattern inventory; correctness and consistency requirements; at least three candidate-model evaluations; selected logical and physical schema; index, partition-key, or graph-traversal design; Go repository adapter implementing conditional version updates and bounded listing; concurrent correctness tests; load tests with tenant skew and hot keys; p50/p95/p99, throughput, error/conflict rate, throttling, storage/index overhead, and recovery measurements; backup/restore rehearsal; and a migration or exit strategy.

**Acceptance criteria:** Demonstrate tenant isolation, atomic version conflict handling, bounded and deterministic pagination, representative skew testing, explicit consistency guarantees, recovery validation, and evidence-based rejection of alternatives. Pending until submitted and reviewed.

### Progress

- **Day:** 11, following the persisted Day 1 through Day 10 lesson history on main.
- **Progression level:** Level 2, Storage and request-path mechanics.
- **Current concept:** SD-011, NoSQL data models and selection.
- **Lesson path:** system-design/lessons/day-11-nosql-data-models-and-selection.md.
- **Prerequisites satisfied:** SD-003 and SD-007.
- **Mini exercise:** Pending learner submission.
- **Substantial exercise:** Pending learner submission.
- **Concepts covered after persistence:** SD-001 through SD-011.
- **Next concept:** SD-012, Object storage and blob lifecycle.
- **Other eligible concepts:** SD-013, Caching fundamentals and cache policies; SD-015, Proxies, reverse proxies, and API gateways; SD-019, Replication fundamentals.
- **Repository update status:** Day 11 lesson, curriculum status, and progress record are committed together on main; the persisted files are read back after the commit.

### Most Important Mental Model

**Select the database model that naturally serves the critical access patterns and correctness contract, then prove the choice under realistic skew, concurrency, failure, and operational recovery.**
