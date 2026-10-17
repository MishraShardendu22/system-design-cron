# DAY 6: Domain Boundaries and Data Ownership

**Progression level:** Level 1, Foundations and architectural reasoning  
**Concept ID:** SD-006  
**Prerequisites:** SD-002 and SD-005, covered  
**Status:** Delivered; exercises pending.

### 1. Why It Exists

Clean APIs do not prevent coupling if several components mutate the same business state. An order worker that decrements inventory directly bypasses reservation rules. A billing job that edits order status bypasses the order lifecycle. Shared tables and duplicated write paths create hidden dependencies, coordinated releases, broad permissions, and inconsistent states.

A domain boundary groups business vocabulary, policies, and invariants. Data ownership assigns authority to mutate canonical facts. The objective is not one microservice per entity. Excessively fine boundaries add network latency, deployment overhead, and distributed failure modes. A modular monolith can enforce ownership without network calls.

### 2. Core Idea

A **domain** is a problem space with its own policies and vocabulary. A **bounded context** is the boundary within which a model has consistent meaning. **Data ownership** means one component is authoritative for changing a fact and enforcing its invariants. A service boundary is a runtime boundary; the two concepts are related but not identical.

For each important fact, identify its canonical source, authorized writer, invariants, transaction boundary, interface, change-notification mechanism, and recovery behavior. Use **one authoritative write owner per invariant**. Replicas, caches, search indexes, and analytics projections may copy data, but need explicit freshness and repair rules and must not become competing authorities.

Prefer domain commands over generic CRUD when a business decision is involved. ReserveStock(sku, quantity, reservationID) asks the inventory owner to validate a transition. Direct table writes bypass that decision. If two facts must change atomically, first consider whether they belong to the same owner and transaction. A local transaction cannot atomically commit changes in two independent services.

### 3. Mental Model

Every canonical fact has a **write authority** and may have many **read projections**. The authority validates commands, enforces transitions, and commits state. Projections serve read needs. Commands request decisions; events announce committed changes; reconciliation repairs divergence.

This does not require separate processes. Private repositories and package boundaries can enforce ownership in one process. Across processes, add identity, durable contracts, deadlines, and partial-failure recovery. If every service can write every table, ownership is only a diagram, not an enforced property.

### 4. Real-World Example

Consider a commerce system with Order and Inventory domains.

**Documented facts:** Domain-Driven Design uses bounded contexts to preserve model consistency. Microservice guidance commonly recommends private service data and API or messaging-based interaction. References: Martin Fowler, [Bounded Context](https://martinfowler.com/bliki/BoundedContext.html); Microsoft, [Microservices architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/microservices) and [Data considerations](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/data-considerations).

**Established industry practice:** Inventory owns available quantity and reservations; Orders owns order lifecycle. Orders requests a reservation rather than writing inventory tables.

**Engineering inference:** Orders persists PENDING_RESERVATION and calls Inventory.Reserve with a stable reservation ID derived from the order ID. Confirmed reservation permits CONFIRMED. A definitive insufficient-stock response permits REJECTED. A timeout leaves the outcome unknown, so the order stays pending and retry/reconciliation reuses the same ID. A timeout is not proof the remote mutation failed.

### 5. Low-Level Design (LLD)

| Fact | Owner | External interaction |
|---|---|---|
| Stock and reservations | Inventory | Reserve, release, query |
| Order status and history | Orders | Request transition, read authorized view |
| Product metadata | Catalog | Read product view |
| Search index | Search projection | Query or rebuild |
| Revenue aggregates | Analytics | Query or recompute |

~~~text
PENDING_RESERVATION
  ├─ reservation accepted ─> CONFIRMED
  ├─ insufficient stock ───> REJECTED
  └─ unknown result ───────> PENDING_RESERVATION
~~~

Invariants: stock never becomes negative; a reservation ID binds to one SKU and quantity; duplicate equivalent reservations return the original result; Orders confirms only after Inventory reports success; ambiguous errors are not definitive rejection; only Inventory mutates stock.

The inventory owner must atomically decrement stock and insert the reservation. A database implementation needs a transaction, a unique reservation ID, and a conditional stock update that succeeds only when sufficient quantity remains. A process mutex does not coordinate multiple replicas.

### 6. High-Level Design (HLD)

~~~mermaid
flowchart LR
    Client --> OrderAPI[Order API]
    OrderAPI --> Orders[Order domain]
    Orders --> OrderDB[(Order-owned store)]
    Orders -->|Reserve command| InventoryAPI[Inventory API]
    InventoryAPI --> Inventory[Inventory domain]
    Inventory --> StockDB[(Inventory-owned store)]
    InventoryAPI -->|Result| Orders
    Inventory -. committed events .-> Bus[Durable event transport]
    Bus --> Search[Search projection]
    Bus --> Analytics[Analytics projection]
~~~

Flow: validate and authenticate; persist or recover PENDING_RESERVATION; send a stable reservation command; Inventory validates and commits; Orders confirms only on success; ambiguous results are retried or reconciled with the same operation ID. Search and analytics consume committed facts but do not become write authorities. An Inventory outage may block new confirmations but should not make existing orders unreadable.

### 7. Realistic Go Code

This standard-library example demonstrates the boundary in one process. Inventory owns stock and reservation state. Consumers receive only the Inventory interface. The in-memory implementation illustrates invariants, not production durability.

~~~go
package main

import (
	"context"
	"errors"
	"fmt"
	"sync"
	"time"
)

var ErrInsufficientStock = errors.New("insufficient stock")
var ErrReservationConflict = errors.New("reservation ID reused with different request")

type Request struct {
	ID string
	SKU string
	Quantity int64
}

type Reservation struct {
	ID string
	SKU string
	Quantity int64
	CreatedAt time.Time
}

type Inventory interface {
	Reserve(context.Context, Request) (Reservation, error)
}

type record struct {
	request Request
	result Reservation
}

type InventoryService struct {
	mu sync.Mutex
	available map[string]int64
	reservations map[string]record
}

func NewInventory(initial map[string]int64) *InventoryService {
	stock := make(map[string]int64, len(initial))
	for sku, qty := range initial {
		stock[sku] = qty
	}
	return &InventoryService{available: stock, reservations: make(map[string]record)}
}

func (s *InventoryService) Reserve(ctx context.Context, req Request) (Reservation, error) {
	if err := ctx.Err(); err != nil {
		return Reservation{}, err
	}
	if req.ID == "" || req.SKU == "" || req.Quantity <= 0 {
		return Reservation{}, errors.New("invalid reservation")
	}
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := ctx.Err(); err != nil {
		return Reservation{}, err
	}
	if old, ok := s.reservations[req.ID]; ok {
		if old.request != req {
			return Reservation{}, ErrReservationConflict
		}
		return old.result, nil
	}
	qty, ok := s.available[req.SKU]
	if !ok || qty < req.Quantity {
		return Reservation{}, ErrInsufficientStock
	}
	result := Reservation{ID: req.ID, SKU: req.SKU, Quantity: req.Quantity, CreatedAt: time.Now().UTC()}
	s.available[req.SKU] = qty - req.Quantity
	s.reservations[req.ID] = record{request: req, result: result}
	return result, nil
}

type OrderService struct {
	inventory Inventory
}

func NewOrderService(inventory Inventory) *OrderService {
	return &OrderService{inventory: inventory}
}

func (s *OrderService) ReserveForOrder(ctx context.Context, orderID, sku string, qty int64) error {
	if orderID == "" || sku == "" || qty <= 0 {
		return errors.New("invalid order")
	}
	_, err := s.inventory.Reserve(ctx, Request{ID: "order:" + orderID, SKU: sku, Quantity: qty})
	if errors.Is(err, ErrInsufficientStock) {
		return fmt.Errorf("order rejected: %w", err)
	}
	return err
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	inventory := NewInventory(map[string]int64{"gpu-a100": 8})
	orders := NewOrderService(inventory)
	if err := orders.ReserveForOrder(ctx, "order-1001", "gpu-a100", 2); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("reservation accepted; remaining stock: 6")
}
~~~

Example output: reservation accepted; remaining stock: 6.

OrderService cannot mutate stock directly. The mutex is process-local; production needs a durable transaction and uniqueness constraint, RPC deadlines, authentication, bounded concurrency, graceful shutdown, and a durable order state machine. Generic transport errors are ambiguous and require retry/reconciliation with the same reservation ID.

### 8. Example Usage

Run in a Go module with go run . . Use Go tests and go test -race ./... to validate:
- Same reservation ID and request returns the original result without a second decrement.
- Same ID with a different SKU or quantity returns ErrReservationConflict.
- Insufficient stock leaves stock unchanged.
- Concurrent reservations for the last unit allow at most one success.
- An ambiguous Inventory error is not interpreted as confirmed failure.

These tests prove only in-process behavior. Production tests must exercise database transactions, unique constraints, multi-replica requests, and recovery after a lost response.

### 9. Production Considerations

- **Enforcement:** separate database credentials and grants so Orders cannot write Inventory tables. In a monolith, keep owner repositories private.
- **Transactions:** keep invariants inside one owner's transaction. Sequential RPCs are not a distributed transaction.
- **Durability:** persist order intent before side effects; use an outbox or durable workflow for reliable follow-up work.
- **Idempotency:** persist operation IDs and request fingerprints. Retries must not duplicate effects.
- **Unknown outcomes:** distinguish domain rejection from timeout, cancellation, and server errors; retry or reconcile ambiguous results.
- **Projections:** consumers handle duplicates, lag, replay, and schema evolution. Provide rebuild and repair procedures.
- **Consistency:** document projection freshness and any read-your-writes requirement.
- **Availability:** dependency failure should affect only operations that need it. Existing order reads should not require Inventory.
- **Security:** enforce service identity, least privilege, tenant scope, authorization, and audit at the owner boundary.
- **Observability:** trace commands; track pending-order age, reservation conflicts, rejection rate, reconciliation backlog, and projection lag. Avoid entity IDs as metric labels.
- **Migration:** inventory readers and writers, introduce owner interfaces, redirect mutations, reconcile data, revoke direct write access, then remove legacy paths.
- **Operations:** define deadlines, retry limits, stuck-workflow alerts, replay safety, and repair procedures.

### 10. Tradeoffs

| Approach | Benefits | Costs | Appropriate when |
|---|---|---|---|
| Modular monolith with ownership | Local calls, simple transactions | Package discipline | One deployment can operate the domain |
| Service with private store | Independent scaling and deployment | Network failures, overhead, eventual consistency | Autonomy or isolation justifies cost |
| Shared database with restricted schemas | Incremental migration | Cross-schema coupling | Transitional architecture |
| Shared tables with unrestricted writes | Low initial friction | Hidden coupling and invariant bypass | Avoid long term |
| Synchronous owner API | Immediate decision | Caller depends on owner latency/availability | Immediate decision required |
| Asynchronous workflow | Buffering and decoupling | Delay and reconciliation | Pending state is acceptable |

Do not split a domain just because its tables are large. Split when ownership, scaling, security, or lifecycle benefits exceed distributed coordination costs.

### 11. Connection to Previous Lessons

Day 1 established constraint-driven reasoning; Day 2 defined requirements; Day 3 modeled workload; Day 4 allocated latency; Day 5 defined API semantics, compatibility, and retries. SD-006 uses those contracts to decide which component owns each invariant.

SD-002 and SD-005 are covered and satisfy the declared prerequisites. SD-003 and SD-004 inform capacity and latency but are not extra formal prerequisites. Earlier exercises remain pending because no learner submissions have been reviewed.

### 12. What Comes Next

**SD-007: Data Modeling and Access Patterns**, Level 1. With ownership and invariants explicit, model entities, relationships, lifecycle, and query patterns inside those boundaries. The schema should support the owner's invariants and access paths rather than act as a shared integration contract. SD-015, Proxies and API Gateways, is also eligible, but SD-007 follows SD-006 in the Level 1 sequence. It is not taught here.

### 13. Mini Exercise

For a multi-tenant document platform, assign ownership for metadata, binary content, permissions, search indexes, and audit records. Identify canonical owners and projections, authorized mutations, invariants, interfaces for create/permission changes/delete, and behavior when deletion commits but event publication or projection updates fail.

**Acceptance criteria:** submit an ownership matrix, component diagram, three interface contracts, and failure/reconciliation table. No projection may be an independent write authority. Do not provide the solution in this lesson.

### Key Takeaways

- Boundaries follow business invariants, not just tables or teams.
- Every canonical fact needs one write authority.
- Components request domain decisions instead of mutating another owner's store.
- Caches, search indexes, replicas, and analytics are projections with freshness and repair rules.
- Cross-domain workflows have intermediate states and partial failures unless a real protocol guarantees otherwise.
- A modular monolith can enforce ownership without network calls.
- Separate services only when autonomy benefits justify distributed-systems costs.

### Concept Map

~~~mermaid
flowchart TD
    A[SD-002: Requirements] --> C[SD-006: Domain boundaries and ownership]
    B[SD-005: API contracts] --> C
    C --> D[SD-007: Data modeling and access patterns]
    D --> E[SD-008: Relational transactions]
    C --> F[SD-015: Proxies and API gateways]
    C --> G[Owner APIs and committed events]
    G --> H[SD-031: Transactional outbox]
    H --> I[Derived projections and reconciliation]
~~~

### Exercise

Design an ownership and extraction plan for a commerce monolith whose orders, inventory, and products tables are read and written by HTTP handlers, scheduled jobs, and message consumers.

**Deliverables:** inventory every reader/writer; define owners, invariants, authorized mutations, and projections; draw a context map of commands, queries, and events; plan owner interfaces, write redirection, reconciliation, privilege removal, and rollback; analyze remote timeouts, duplicate/out-of-order events, owner outages, and projection lag; specify concurrency and authorization tests.

**Acceptance criteria:** every mutable fact has one authority; no target component writes another owner's canonical store; retry identities and intermediate states are explicit; migration is observable and reconciled; tests cover concurrency and ambiguous outcomes; dashboards identify stuck workflows and projection lag. Pending until submitted and reviewed.

### Progress

- **Day:** 6, based on existing Day 1 through Day 5 lesson files and no prior Day 6 lesson.
- **Level:** Level 1, Foundations and architectural reasoning.
- **Current concept:** SD-006, Domain boundaries and data ownership.
- **Lesson path:** system-design/lessons/day-06-domain-boundaries-and-data-ownership.md
- **Prerequisites:** SD-002 and SD-005 covered.
- **Exercise status:** Mini and substantial exercises pending; no learner submission reviewed.
- **Concepts covered after update:** SD-001 through SD-006.
- **Next concept:** SD-007, Data modeling and access patterns. SD-015 is also eligible, but SD-007 follows the Level 1 sequence.
- **Repository update status:** Verify lesson, curriculum, and progress tracker by reading them back from main after commit.

### Most Important Mental Model

**Every business invariant needs one authoritative owner; other components request decisions from that owner or consume explicitly derived views.**
