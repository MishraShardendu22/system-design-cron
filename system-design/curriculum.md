# System Design Curriculum

Statuses: **covered** means a lesson exists; **eligible** means all declared prerequisites are covered; **planned** means prerequisites remain. Exercise completion is tracked separately in progress.md.

## Level 1: Foundations and architectural reasoning

| ID | Concept | Prerequisites | Learning objectives | Status |
|---|---|---|---|---|
| SD-001 | System design fundamentals and architectural reasoning | None | Define boundaries, reason from constraints, identify quality attributes, and make tradeoffs explicit. | covered |
| SD-002 | Requirements discovery and specification | SD-001 | Separate functional and non-functional requirements, constraints, assumptions, and out-of-scope behavior. | covered |
| SD-003 | Capacity estimation and workload modeling | SD-002 | Estimate request rates, storage growth, bandwidth, concurrency, and peak ratios. | covered |
| SD-004 | Latency budgets and performance objectives | SD-002, SD-003 | Decompose end-to-end latency and reason about percentiles, queueing, and budgets. | covered |
| SD-005 | API design, contracts, and versioning | SD-002 | Define semantics, idempotency, pagination, errors, compatibility, and evolution. | covered |
| SD-006 | Domain boundaries and data ownership | SD-002, SD-005 | Assign ownership, define invariants, and identify service boundaries. | covered |
| SD-007 | Data modeling and access patterns | SD-003, SD-006 | Model entities, lifecycle, invariants, and query-driven schemas. | covered |

## Level 2: Storage and request-path mechanics

| ID | Concept | Prerequisites | Learning objectives | Status |
|---|---|---|---|---|
| SD-008 | Relational databases and transaction semantics | SD-007 | Apply constraints, isolation, transactions, and atomicity. | covered |
| SD-009 | Index structures and query planning | SD-008 | Understand B-trees, selectivity, query plans, write amplification, and index tradeoffs. | covered |
| SD-010 | Query optimization and connection pooling | SD-008, SD-009 | Diagnose slow queries and bound database concurrency. | covered |
| SD-011 | NoSQL data models and selection | SD-007, SD-003 | Compare key-value, document, wide-column, and graph models against access patterns. | covered |
| SD-012 | Object storage and blob lifecycle | SD-007 | Design large-object storage, metadata, retention, and transfer flows. | eligible |
| SD-013 | Caching fundamentals and cache policies | SD-004, SD-007 | Choose keys, TTLs, eviction, and invalidation strategies. | eligible |
| SD-014 | Cache consistency, stampedes, and penetration | SD-013 | Handle stale data, hot keys, miss storms, and negative caching. | planned |
| SD-015 | Proxies, reverse proxies, and API gateways | SD-005 | Place routing, TLS termination, policy enforcement, and transformation. | eligible |
| SD-016 | Load balancing and health-based routing | SD-003, SD-015 | Compare routing algorithms, health checks, and overload behavior. | planned |
| SD-017 | CDNs and edge delivery | SD-012, SD-013, SD-015 | Design cacheability, invalidation, origin shielding, and edge security. | planned |

## Level 3: Scaling and distributed state

| ID | Concept | Prerequisites | Learning objectives | Status |
|---|---|---|---|---|
| SD-018 | Vertical scaling, horizontal scaling, and bottleneck analysis | SD-003, SD-016 | Identify scaling limits across compute, memory, network, and storage. | planned |
| SD-019 | Replication fundamentals | SD-008, SD-011 | Compare primary-replica and multi-writer designs, lag, and failover. | eligible |
| SD-020 | Consistency models and read/write guarantees | SD-019 | Distinguish linearizability, sequential, causal, and eventual consistency. | planned |
| SD-021 | Partitioning and sharding strategies | SD-003, SD-007, SD-018 | Choose partition keys, manage skew, and reason about cross-shard operations. | planned |
| SD-022 | Resharding and data rebalancing | SD-021 | Move partitions safely while maintaining availability. | planned |
| SD-023 | Distributed transactions and sagas | SD-008, SD-019, SD-020 | Compare 2PC, sagas, compensations, and cross-service invariants. | planned |
| SD-024 | Time, ordering, and idempotency | SD-005, SD-019 | Handle retries, duplicates, clock uncertainty, and ordering constraints. | planned |
| SD-025 | Failure models, partial failure, and timeouts | SD-016, SD-019 | Model crash, omission, delay, partition, and correlated failures. | planned |
| SD-026 | Leases, locks, and fencing tokens | SD-020, SD-024, SD-025 | Prevent stale owners and reason about expiry and split brain. | planned |
| SD-027 | Coordination, leader election, and consensus | SD-020, SD-025, SD-026 | Explain quorum, epochs, safety, liveness, and consensus boundaries. | planned |
| SD-028 | Membership, service discovery, and configuration | SD-016, SD-025 | Track endpoints and configuration under churn and partial failure. | planned |

## Level 4: Messaging and data movement

| ID | Concept | Prerequisites | Learning objectives | Status |
|---|---|---|---|---|
| SD-029 | Queues and asynchronous work distribution | SD-005, SD-024, SD-025 | Decouple producers and consumers, bound work, and handle retries. | planned |
| SD-030 | Delivery semantics and deduplication | SD-024, SD-029 | Distinguish at-most-once, at-least-once, and effectively-once processing. | planned |
| SD-031 | Event-driven architecture and transactional outbox | SD-008, SD-024, SD-029 | Publish durable events without dual-write loss. | planned |
| SD-032 | Pub/sub, ordering, and consumer groups | SD-029, SD-030 | Partition streams and define ordering and fan-out guarantees. | planned |
| SD-033 | Stream processing and stateful operators | SD-030, SD-032 | Handle windows, watermarks, late data, checkpoints, and recovery. | planned |
| SD-034 | CDC, replay, and schema evolution | SD-008, SD-031, SD-032 | Capture database changes, replay safely, and evolve event contracts. | planned |
| SD-035 | Workflow orchestration and durable execution | SD-023, SD-029, SD-030 | Model long-running workflows, retries, compensation, and durable state. | planned |

## Level 5: Reliability, operations, and security

| ID | Concept | Prerequisites | Learning objectives | Status |
|---|---|---|---|---|
| SD-036 | Timeouts, retries, backoff, and jitter | SD-025, SD-029 | Prevent retry amplification and define retry budgets. | planned |
| SD-037 | Circuit breakers, bulkheads, and load shedding | SD-018, SD-025, SD-036 | Bound failure propagation and preserve critical work under overload. | planned |
| SD-038 | Graceful degradation and fallback design | SD-013, SD-025, SD-037 | Define degraded modes, stale reads, and feature priorities. | planned |
| SD-039 | Redundancy, failover, backup, and disaster recovery | SD-019, SD-025, SD-027 | Design recovery objectives, failover, backups, and restore validation. | planned |
| SD-040 | Reliability engineering and fault injection | SD-037, SD-039 | Validate failure assumptions and quantify resilience. | planned |
| SD-041 | Observability: logs, metrics, traces, and profiling | SD-004, SD-016 | Diagnose bottlenecks and instrument service boundaries. | planned |
| SD-042 | SLIs, SLOs, error budgets, and alerting | SD-041 | Define user-centric objectives and actionable alerts. | planned |
| SD-043 | Capacity planning, autoscaling, and admission control | SD-003, SD-018, SD-037, SD-041 | Forecast demand and scale without destabilizing dependencies. | planned |
| SD-044 | Incident response, postmortems, and operational readiness | SD-039, SD-042 | Establish response, learning, and measurable follow-up. | planned |
| SD-045 | Authentication, authorization, and service identity | SD-005, SD-015 | Apply identity, least privilege, and credential lifecycle controls. | planned |
| SD-046 | Threat modeling, secrets, and data protection | SD-045, SD-012 | Identify trust boundaries and protect sensitive data. | planned |
| SD-047 | Multi-tenancy, isolation, and noisy-neighbor control | SD-021, SD-037, SD-045 | Enforce tenant isolation, quotas, and fairness. | planned |
| SD-048 | Deployment strategies and configuration safety | SD-028, SD-039, SD-041 | Use progressive delivery, rollback, and safe configuration rollout. | planned |
| SD-049 | Infrastructure as code and platform boundaries | SD-041, SD-045, SD-048 | Define reproducible infrastructure and environment promotion. | planned |
| SD-050 | Cost modeling and efficiency tradeoffs | SD-003, SD-041, SD-043 | Attribute cost and compare cost, performance, and reliability. | planned |

## Level 6: Architecture synthesis and advanced systems

| ID | Concept | Prerequisites | Learning objectives | Status |
|---|---|---|---|---|
| SD-051 | Search and indexing architecture | SD-007, SD-011, SD-031 | Design indexing pipelines, freshness, ranking, and serving. | planned |
| SD-052 | Rate limiting and quota enforcement | SD-005, SD-020, SD-029 | Compare token bucket, leaky bucket, fixed window, and distributed quotas. | planned |
| SD-053 | Feed generation and fan-out strategies | SD-013, SD-021, SD-032 | Compare fan-out-on-write and fan-out-on-read. | planned |
| SD-054 | Object upload, media processing, and delivery | SD-012, SD-017, SD-029, SD-046 | Design secure ingestion, asynchronous processing, and edge delivery. | planned |
| SD-055 | URL shortening and key-generation systems | SD-005, SD-008, SD-021, SD-052 | Design identifiers, redirects, and abuse controls. | planned |
| SD-056 | Notification delivery platform | SD-029, SD-030, SD-036, SD-045 | Design preferences, provider routing, retries, and delivery tracking. | planned |
| SD-057 | Distributed cache and key-value service | SD-014, SD-019, SD-020, SD-021, SD-027 | Combine partitioning, replication, eviction, and failure handling. | planned |
| SD-058 | Metrics ingestion and time-series storage | SD-033, SD-041, SD-043, SD-052 | Design high-cardinality ingestion, aggregation, retention, and queries. | planned |
| SD-059 | Log ingestion and analytical data platform | SD-033, SD-034, SD-041, SD-043 | Build durable ingestion, partitioned storage, replay, and analytics. | planned |
| SD-060 | Real-time collaboration and presence | SD-020, SD-024, SD-027, SD-032 | Design concurrent updates, presence expiry, and conflict resolution. | planned |
| SD-061 | Durable job scheduler | SD-024, SD-027, SD-029, SD-036 | Design claiming, leases, retries, and recovery. | planned |
| SD-062 | Payment and order processing | SD-008, SD-023, SD-024, SD-030, SD-045 | Preserve invariants across retries, providers, and workflows. | planned |
| SD-063 | Multi-region architecture and data residency | SD-020, SD-027, SD-039, SD-046, SD-047 | Choose regional topology and manage regional failure. | planned |
| SD-064 | Migration, backfill, and online schema evolution | SD-008, SD-021, SD-031, SD-048 | Execute compatible migrations, reconciliation, and cutover. | planned |
| SD-065 | End-to-end design review and architecture decision records | SD-004, SD-005, SD-018, SD-020, SD-037, SD-042, SD-050 | Synthesize requirements, estimates, architecture, risks, and alternatives. | planned |

## Dependency policy

This is a prerequisite DAG, not a rigid linear checklist. A concept is eligible when all listed prerequisites are covered. Select the earliest useful eligible concept unless repository evidence justifies another order. Each lesson teaches one primary concept only.
