# DAY 10: Query Optimization and Connection Pooling

**Progression level:** Level 2, Storage and request-path mechanics  
**Concept ID:** SD-010  
**Prerequisites:** SD-008 and SD-009, covered.  
**Status:** Delivered; exercises pending.

### 1. Why It Exists

Correct SQL can be expensive: it may scan many rows to return a few, sort large intermediate results, repeat lookups in an N+1 pattern, fetch unused columns, or rely on stale estimates. Requests can also queue while waiting for database connections. A request taking 400 ms might spend 330 ms waiting for a connection and 70 ms executing SQL. Query tuning alone will not remove pool wait. Increasing the pool can worsen database contention if CPU, I/O, memory, or locks are already saturated.

The goal is to reduce unnecessary work per request and bound concurrency to what the database can sustain, rather than blindly adding indexes or connections.

### 2. Core Idea

Use an evidence-driven loop: identify an expensive query, capture representative parameters, inspect its plan, compare estimated and actual rows/loops, locate the dominant cost, change one variable, and measure correctness, latency, throughput, and write-side effects.

PostgreSQL's planner chooses a physical plan from statistics and cost estimates. EXPLAIN displays the plan; EXPLAIN ANALYZE executes the statement and adds actual observations. See [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) and [planner statistics](https://www.postgresql.org/docs/current/planner-stats.html).

Inspect estimate errors, loops, rows removed by filters, sorts, temporary I/O, buffers, join strategy, and result volume. A sequential scan can be optimal for small tables or broad reads. An index scan can be slower when it causes many random heap reads. Verify plans using realistic data and parameters.

In Go, *sql.DB is a concurrency-safe handle managing a pool, not a single connection. SetMaxOpenConns limits open connections and callers may wait at that limit. DB.Stats exposes wait counters. See [Go connection management](https://go.dev/doc/database/manage-connections) and [database/sql](https://pkg.go.dev/database/sql).

Pool size controls concurrency, not guaranteed throughput. Budget connections across all replicas, workers, migrations, monitoring, and administrative clients. More connections help only while the database has capacity.

### 3. Mental Model

~~~text
request -> application scheduling -> pool wait -> DB lock/execution
        -> result transfer and scan -> response
~~~

Measure pool acquisition separately from execution. Little's Law gives the steady-state relation L = arrival rate × time in system. At 250 operations/s and 40 ms average connection-holding time, the mean busy-connection count is about 10, not a final pool-size recommendation. Bursts, variance, lock contention, and headroom matter. Queueing grows rapidly near saturation.

### 4. Real-World Example

A multi-tenant event API returns the newest 50 summaries from hundreds of millions of rows.

**Documented facts:** PostgreSQL documents plan inspection, statistics, and index behavior; Go documents pool controls. References: [EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html), [indexes](https://www.postgresql.org/docs/current/indexes.html), and [Go connection management](https://go.dev/doc/database/manage-connections).

**Established industry practice:** use query telemetry, narrow projections, bounded result sets, parameterized SQL, short transactions, and separate pool-wait metrics.

**Engineering inference:** tenant equality followed by a time range and newest-first ordering suggests testing a composite B-tree on tenant_id, created_at, and event_id. Including event_type may help a narrow projection, but the executed plan must validate the hypothesis. This is illustrative, not a claim about a proprietary system.

Forty replicas each allowing 40 open connections create a configured ceiling of 1,600 before other clients. Derive pool limits from the shared database budget and measurements, not HTTP worker count.

### 5. Low-Level Design (LLD)

~~~sql
CREATE TABLE events (
    event_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL
);

CREATE INDEX CONCURRENTLY events_tenant_created_id_idx
    ON events (tenant_id, created_at DESC, event_id DESC)
    INCLUDE (event_type);
~~~

The index supports tenant equality, time filtering, deterministic order, and the narrow projection. Do not include a large payload without evidence because it increases index size and write amplification. CREATE INDEX CONCURRENTLY has caveats, cannot run in a transaction block, and can leave an invalid index after failure. See [PostgreSQL CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html).

First page:

~~~sql
SELECT event_id, created_at, event_type
FROM events
WHERE tenant_id = $1 AND created_at >= $2 AND created_at < $3
ORDER BY created_at DESC, event_id DESC
LIMIT $4;
~~~

Continuation page adds the keyset predicate (created_at, event_id) < ($4, $5). Separate statements avoid an optional-cursor OR expression that can complicate estimates. Bind values and allowlist dynamic SQL structure.

Configure one shared pool at startup, ping under a deadline, bound idle connections, and reuse the same *sql.DB. Coordinate request deadlines with server-side statement and lock timeouts.

### 6. High-Level Design (HLD)

~~~mermaid
flowchart TD
  Client --> API[Validate tenant, range, cursor, limit]
  API --> Deadline[Request deadline]
  Deadline --> Pool[Shared Go pool]
  Pool --> Wait[Connection wait]
  Wait --> DB[(PostgreSQL)]
  DB --> Plan[Planner and execution]
  Plan --> Rows[Bounded result]
  Rows --> API
  Pool --> PM[Pool metrics]
  DB --> DM[Query, lock and I/O metrics]
  PM --> Obs[Correlated diagnostics]
  DM --> Obs
~~~

The API validates authorized tenant identity and bounded input. The repository uses context-aware parameterized SQL. Correlate normalized query fingerprints with pool wait, execution, lock waits, and resource utilization. If query latency rises, connections stay occupied longer, pool wait increases, and retries can amplify load. Sustained overload requires less work, admission limits, load shedding, or actual database capacity.

### 7. Realistic Go Code

A PostgreSQL driver must be registered by the application. Credentials come from deployment configuration.

~~~go
package eventstore

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "strings"
    "time"
)

var ErrInvalidQuery = errors.New("invalid event query")

type PoolConfig struct {
    MaxOpenConns int
    MaxIdleConns int
    ConnMaxLifetime time.Duration
    ConnMaxIdleTime time.Duration
}

func Open(ctx context.Context, driverName, dsn string, c PoolConfig) (*sql.DB, error) {
    if ctx == nil || strings.TrimSpace(driverName) == "" || strings.TrimSpace(dsn) == "" {
        return nil, errors.New("context, driver and DSN required")
    }
    if c.MaxOpenConns < 1 || c.MaxIdleConns < 0 ||
        c.MaxIdleConns > c.MaxOpenConns || c.ConnMaxLifetime < 0 ||
        c.ConnMaxIdleTime < 0 {
        return nil, errors.New("invalid pool configuration")
    }
    db, err := sql.Open(driverName, dsn)
    if err != nil { return nil, fmt.Errorf("open database: %w", err) }
    db.SetMaxOpenConns(c.MaxOpenConns)
    db.SetMaxIdleConns(c.MaxIdleConns)
    db.SetConnMaxLifetime(c.ConnMaxLifetime)
    db.SetConnMaxIdleTime(c.ConnMaxIdleTime)
    if err := db.PingContext(ctx); err != nil {
        _ = db.Close()
        return nil, fmt.Errorf("ping database: %w", err)
    }
    return db, nil
}

type Event struct {
    ID int64
    CreatedAt time.Time
    EventType string
}
type Cursor struct { CreatedAt time.Time; ID int64 }
type Store struct { DB *sql.DB }

func validateList(tenant string, from, to time.Time, c *Cursor, limit int) error {
    if strings.TrimSpace(tenant) == "" || from.IsZero() || to.IsZero() ||
        !from.Before(to) || limit < 1 || limit > 100 {
        return ErrInvalidQuery
    }
    if c != nil && (c.CreatedAt.IsZero() || c.ID <= 0 ||
        c.CreatedAt.Before(from) || !c.CreatedAt.Before(to)) {
        return ErrInvalidQuery
    }
    return nil
}

func (s Store) ListRecent(ctx context.Context, tenant string, from, to time.Time,
    c *Cursor, limit int) ([]Event, error) {
    if ctx == nil || s.DB == nil { return nil, errors.New("context and DB required") }
    if err := validateList(tenant, from, to, c, limit); err != nil { return nil, err }

    var rows *sql.Rows
    var err error
    if c == nil {
        rows, err = s.DB.QueryContext(ctx,
            "SELECT event_id, created_at, event_type FROM events "+
                "WHERE tenant_id=$1 AND created_at >= $2 AND created_at < $3 "+
                "ORDER BY created_at DESC, event_id DESC LIMIT $4",
            tenant, from, to, limit)
    } else {
        rows, err = s.DB.QueryContext(ctx,
            "SELECT event_id, created_at, event_type FROM events "+
                "WHERE tenant_id=$1 AND created_at >= $2 AND created_at < $3 "+
                "AND (created_at, event_id) < ($4, $5) "+
                "ORDER BY created_at DESC, event_id DESC LIMIT $6",
            tenant, from, to, c.CreatedAt, c.ID, limit)
    }
    if err != nil { return nil, fmt.Errorf("query events: %w", err) }
    defer rows.Close()

    out := make([]Event, 0, limit)
    for rows.Next() {
        var e Event
        if err := rows.Scan(&e.ID, &e.CreatedAt, &e.EventType); err != nil {
            return nil, fmt.Errorf("scan event: %w", err)
        }
        out = append(out, e)
    }
    if err := rows.Err(); err != nil { return nil, fmt.Errorf("iterate events: %w", err) }
    return out, nil
}
~~~

The shared pool is initialized once. The query method bounds range and page size, propagates cancellation, closes rows, and checks iteration errors. Tenant identity must come from authorization context and the cursor should be opaque and integrity-protected. Production code still needs tracing, normalized query metrics, driver-specific error classification, server-side timeout policy, graceful shutdown, and PostgreSQL integration tests. Do not tune the pool per request.

### 8. Example Usage

~~~go
ctx, cancel := context.WithTimeout(request.Context(), 250*time.Millisecond)
defer cancel()
page, err := store.ListRecent(ctx, authenticatedTenantID,
    rangeStart, rangeEnd, cursor, 50)
if err != nil { return err }

var next *eventstore.Cursor
if len(page) > 0 {
    last := page[len(page)-1]
    next = &eventstore.Cursor{CreatedAt: last.CreatedAt, ID: last.ID}
}
_ = next
~~~

Serialize a next cursor only for a nonempty page. Equal timestamps remain deterministic due to event_id. Keyset pagination avoids large-offset discard work but does not freeze the dataset across requests.

Unit tests should cover valid input, empty tenant, invalid range, limit above 100, and malformed cursor. PostgreSQL integration tests should verify adjacent-page ordering, cancellation, pool saturation, and result equivalence. Compare p50/p95/p99, throughput, rows scanned, buffers, temporary I/O, and pool wait under concurrency.

### 9. Production Considerations

- Track request latency, pool wait, execution, lock waits, row counts, and result bytes separately.
- Use representative tenant skew, ranges, cache warmth, and concurrency.
- Compare estimated and actual rows; refresh statistics only when evidence supports it.
- Remove N+1 calls, excess columns, unbounded reads, and avoidable round trips.
- Test parameter-sensitive plans when tenant sizes vary.
- Budget connections across replicas, workers, migrations, and rollout overlap.
- Export deltas of cumulative WaitCount and WaitDuration with open, in-use, and idle connections.
- High wait with idle DB resources suggests pool/configuration or connection-retention issues. High wait with saturated resources suggests reducing work or concurrency.
- Coordinate request deadlines with statement and lock timeouts.
- Avoid remote calls inside transactions; long transactions pin connections.
- Test graceful shutdown, failover, and connection resets.
- External poolers can multiplex connections but change session-state assumptions.
- Parameterize values, allowlist dynamic SQL structure, authorize tenants, and redact sensitive parameters.
- Include index storage, WAL, write latency, and migration risk in the decision.
- EXPLAIN ANALYZE executes statements, so use controlled environments for mutations.
- Evaluate tail latency and bursts, not averages alone.

### 10. Tradeoffs

| Choice | Benefit | Cost or limitation | Appropriate use |
|---|---|---|---|
| Query-shape improvement | Removes work without storage | Code changes may be needed | N+1 or excess data |
| Composite B-tree | Selective filtering/order | Write and storage cost | Frequent equality/range/order pattern |
| Covering index | May reduce heap reads | Larger index; visibility still matters | Narrow repeated projection |
| Sequential scan | Efficient broad read | Scales with relation size | Small tables or broad result |
| Larger pool | More concurrency with headroom | More contention under saturation | Measured pool bottleneck |
| Smaller pool | Bounds DB concurrency | More application queueing | Protect shared DB |
| Offset pagination | Simple page numbers | Increasing discard work | Small result sets |
| Keyset pagination | Efficient continuation | Requires deterministic keys | Large feeds |
| External pooler | Multiplexes clients | Extra operations and session constraints | Many short-lived clients |

The best pool size is the smallest tested limit that sustains target throughput and latency with headroom.

### 11. Connection to Previous Lessons

Day 8, SD-008, established transactions, isolation, and locking. Day 9, SD-009, established indexes, selectivity, and plan interpretation. SD-010 applies these foundations to query cost and connection concurrency. SD-003 supplies workload estimates, SD-004 latency budgets, and SD-007 the schema and access patterns. Prior exercises remain pending because no submissions have been reviewed; they do not block this concept.

### 12. What Comes Next

**SD-011: NoSQL Data Models and Selection**, Level 2. It compares key-value, document, wide-column, and graph models against access patterns, consistency needs, and operational constraints. Its prerequisites, SD-007 and SD-003, are covered. SD-012, SD-013, and SD-015 are also eligible, but SD-011 is next in curriculum order. It is not taught here.

### 13. Mini Exercise

A multi-tenant endpoint's p99 rises from 90 ms to 650 ms after traffic doubles. Traces show 420 ms waiting for a connection. PostgreSQL reports high CPU and a plan returning 100 rows after scanning 900,000. The pool limit is 60 connections per instance across 20 instances.

Propose a diagnostic and remediation sequence separating pool wait from query execution. State what evidence would justify reducing concurrency, changing SQL/indexes, refreshing statistics, or adding capacity, and how to verify read and write effects.

**Acceptance criteria:** Calculate the aggregate connection ceiling, distinguish wait from execution, identify plan evidence, propose controlled changes, measure read/write impact, and define rollback criteria. Do not provide the full solution.

### Key Takeaways

- Optimize from observed plans and representative workloads.
- Compare estimated/actual rows, loops, buffers, sorts, and result volume.
- Reduce unnecessary rows, columns, round trips, and repeated queries first.
- Measure pool wait separately from query execution.
- Aggregate connection budgets across replicas and other clients.
- More connections can reduce throughput under saturation.
- Bound page sizes, deadlines, and transaction duration.
- Validate tail latency, correctness, write costs, and operations.

### Concept Map

~~~mermaid
flowchart TD
    A[SD-003 Workload estimates] --> D[SD-010 Query optimization and pooling]
    B[SD-004 Latency budgets] --> D
    C[SD-007 Data modeling] --> E[SD-008 Transactions]
    E --> F[SD-009 Indexes and plans]
    F --> D
    D --> G[Plan diagnosis]
    D --> H[Pool sizing and queueing]
    D --> I[Deadlines and bounded concurrency]
    D --> J[SD-011 NoSQL models]
~~~

### Exercise

Design and benchmark an event-history read path.

**Deliverables:** PostgreSQL schema and skewed data generator; baseline SQL and plans; Go repository using a shared *sql.DB, deadlines, keyset pagination, and configurable pool limits; load tests varying request rate, tenant skew, and pool size; before/after p50/p95/p99, throughput, error rate, pool wait, rows scanned, buffers, and write throughput; global connection-budget calculation; and a decision record with rejected alternatives and rollback thresholds.

**Acceptance criteria:** Deterministic pagination, tenant isolation, equivalent results, cancellation and pool-saturation tests, reproducible measurements, explicit connection budget, and evidence for performance claims. Pending until submitted and reviewed.

### Progress

- **Day:** 10, based on Day 1 through Day 9 lesson history on main.
- **Progression level:** Level 2, Storage and request-path mechanics.
- **Current concept:** SD-010, Query optimization and connection pooling.
- **Lesson path:** system-design/lessons/day-10-query-optimization-and-connection-pooling.md.
- **Prerequisites satisfied:** SD-008 and SD-009.
- **Mini exercise:** Pending.
- **Substantial exercise:** Pending.
- **Concepts covered after persistence:** SD-001 through SD-010.
- **Next concept:** SD-011, NoSQL data models and selection.
- **Other eligible concepts:** SD-012, SD-013, and SD-015.
- **Repository update status:** Read the lesson, curriculum, and progress back from main before claiming completion.

### Most Important Mental Model

**Reduce unnecessary database work, then bound concurrency to what the database can sustain, measuring query execution and connection-pool waiting separately.**
