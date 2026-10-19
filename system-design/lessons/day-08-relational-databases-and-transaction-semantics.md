# DAY 8: Relational Databases and Transaction Semantics

**Progression level:** Level 2, Storage and request-path mechanics  
**Concept ID:** SD-008  
**Prerequisites:** SD-007, Data modeling and access patterns, covered by Day 7  
**Status:** Lesson delivered; exercises pending.

### 1. Why It Exists

Concurrent requests can read and mutate the same state, processes can fail between SQL statements, and clients can lose the connection while commit is being acknowledged. Correct individual queries do not guarantee a correct business operation. A transfer that debits one account and fails before crediting another violates the invariant. Two inventory reservations can oversell stock if both perform an unprotected read-check-write sequence. A retry after timeout can duplicate a mutation.

Transactions define an atomic database unit. Isolation and locks constrain concurrent histories. Constraints enforce legal states. Commit defines the database success boundary, but a lost acknowledgement can leave the caller uncertain whether the transaction committed.

### 2. Core Idea

ACID describes four dimensions: **atomicity**, all transaction writes commit or roll back together; **consistency**, committed work preserves enforced constraints and encoded invariants; **isolation**, concurrency is restricted according to the selected level; **durability**, an acknowledged commit survives failures covered by the database's documented contract. Configuration, storage, replication, and acknowledgement policy affect the actual durability guarantee.

Atomicity does not imply serializable execution. A local transaction does not atomically include HTTP calls, object storage, or a message broker. Cross-system side effects require another protocol, such as a transactional outbox or saga.

Isolation levels are engine-specific. PostgreSQL treats Read Uncommitted as Read Committed. At Read Committed, each statement gets a snapshot as of that statement's start, so statements in one transaction may see different commits. PostgreSQL Repeatable Read provides a stable snapshot but can still permit serialization anomalies. Serializable guarantees an outcome equivalent to a serial execution, but a transaction may abort and must be retried as a whole. See the [PostgreSQL isolation guide](https://www.postgresql.org/docs/current/transaction-iso.html).

Constraints reject illegal states; row locks serialize conflicting mutations; optimistic versions reject stale writes. A **deadlock** is a cycle of transactions waiting for each other's locks. Databases usually abort one participant. Consistent lock order reduces common cycles.

A **commit outcome ambiguity** occurs when the database may have committed but the acknowledgement did not reach the client. A timeout is not proof of rollback. Stable operation IDs and durable idempotency records let a retry discover whether the operation already succeeded.

### 3. Mental Model

Treat a transaction as a proposed state transition guarded by database rules. It reads state, obtains conflict protection, writes changes, and asks the database to commit.

For a transfer, the invariant is that debit, credit, and transfer record agree for every committed operation, and neither balance becomes negative.

~~~text
BEGIN -> claim transfer ID -> lock accounts in stable order
      -> validate funds -> debit -> credit -> mark committed -> COMMIT
~~~

A definite pre-commit error causes rollback. A successful commit establishes the database result. A lost commit acknowledgement creates an unknown outcome. Retry with the same transfer ID so the uniqueness constraint resolves duplicate delivery.

Locks do not protect rows that were not locked; snapshot isolation does not prevent every write-skew anomaly; primary commit does not guarantee immediate cache, search-index, or asynchronous-replica freshness.

### 4. Real-World Example

A ledger service transfers integer cents between PostgreSQL account rows.

**Documented facts:** PostgreSQL documents transaction isolation, row-level locking, constraints, and deadlock behavior. Go's `database/sql` exposes `BeginTx`, `Commit`, `Rollback`, and context-aware queries. References: [PostgreSQL locking](https://www.postgresql.org/docs/current/explicit-locking.html), [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html), and [Go database/sql](https://pkg.go.dev/database/sql).

**Established industry practice:** Keep related ledger mutations in one transaction, use database constraints for critical invariants, use integer minor currency units, bound transaction duration, and define retry semantics.

**Engineering inference:** One relational database is a reasonable baseline when both accounts share one transactional owner. If accounts later live in independent services or databases, this local transaction no longer protects the cross-boundary invariant. This is a hypothetical example, not a claim about a proprietary payment system.

### 5. Low-Level Design (LLD)

~~~sql
CREATE TABLE accounts (
  account_id UUID PRIMARY KEY,
  balance_cents BIGINT NOT NULL CHECK (balance_cents >= 0)
);
CREATE TABLE transfers (
  transfer_id UUID PRIMARY KEY,
  from_account_id UUID NOT NULL REFERENCES accounts(account_id),
  to_account_id UUID NOT NULL REFERENCES accounts(account_id),
  amount_cents BIGINT NOT NULL CHECK (amount_cents > 0),
  status TEXT NOT NULL CHECK (status IN ('pending', 'committed')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (from_account_id <> to_account_id)
);
~~~

The transfer primary key is the deduplication boundary. Foreign keys reject missing accounts; checks reject invalid amounts, negative balances, and same-account transfers. Debit, credit, and transfer status must change in one transaction. Here, pending is inserted and changed to committed within the same transaction, so rollback removes the insert. A durable externally visible pending state would require a separate recoverable state machine.

Lock the two account rows one at a time in sorted ID order. Validate balances after locks are acquired. Never hold locks during remote calls. A unique transfer ID must remain stable across retries, and reusing it with a different payload must be rejected.

Retry only classified transient failures such as deadlock or serialization errors, using a fresh transaction and bounded attempts. Do not retry business rejections. A commit error may be ambiguous, so retry or query with the same operation ID. Classify driver errors by SQLSTATE, not message text.

### 6. High-Level Design (HLD)

~~~mermaid
flowchart TD
  Client --> API[Transfer API]
  API --> Auth[Authenticate and authorize]
  Auth --> Validate[Validate amount and stable ID]
  Validate --> Tx[Transaction coordinator]
  Tx --> DB[(PostgreSQL primary)]
  DB --> A[Accounts and constraints]
  DB --> T[Unique transfer records]
  Tx --> Result[Commit result or unknown outcome]
  Result --> Client
~~~

The API authenticates and authorizes both accounts, validates input, and establishes a deadline. The transaction claims the transfer ID, checks an existing record on conflict, locks both accounts, validates funds, updates both balances, marks the transfer committed, and commits. Return success after commit acknowledgement. If acknowledgement fails, require retry with the same ID.

The database is authoritative. A cache or replica must not validate spendable funds. If a transfer must emit an event, write event intent to an outbox in the same transaction and publish asynchronously; consumers must tolerate duplicates.

Hot account rows, transaction duration, database write capacity, and pool capacity bound throughput. Adding application replicas does not remove serialization required by shared invariants. Sharding accounts changes the atomicity boundary.

### 7. Realistic Go Code

This uses the standard library and PostgreSQL-style SQL. Register a PostgreSQL driver and validate UUID syntax at the API boundary.

~~~go
package ledger

import (
  "context"
  "database/sql"
  "errors"
  "fmt"
  "math"
  "sort"
)

var (
  ErrInvalidTransfer = errors.New("invalid transfer")
  ErrAccountNotFound = errors.New("account not found")
  ErrInsufficientFunds = errors.New("insufficient funds")
  ErrIdempotencyConflict = errors.New("transfer ID payload mismatch")
  ErrCommitOutcomeUnknown = errors.New("commit outcome unknown")
)

type TransferRequest struct {
  ID, FromAccountID, ToAccountID string
  AmountCents int64
}

type CommitUnknown struct{ Cause error }
func (e *CommitUnknown) Error() string { return fmt.Sprintf("%v: %v; retry with same ID", ErrCommitOutcomeUnknown, e.Cause) }
func (e *CommitUnknown) Unwrap() error { return e.Cause }
func (e *CommitUnknown) Is(target error) bool { return target == ErrCommitOutcomeUnknown }

func lockBalance(ctx context.Context, tx *sql.Tx, id string) (int64, error) {
  var balance int64
  err := tx.QueryRowContext(ctx,
    "SELECT balance_cents FROM accounts WHERE account_id = $1 FOR UPDATE", id,
  ).Scan(&balance)
  if errors.Is(err, sql.ErrNoRows) { return 0, ErrAccountNotFound }
  if err != nil { return 0, fmt.Errorf("lock account: %w", err) }
  return balance, nil
}

func Transfer(ctx context.Context, db *sql.DB, r TransferRequest) error {
  if ctx == nil || db == nil { return errors.New("context and database required") }
  if r.ID == "" || r.FromAccountID == "" || r.ToAccountID == "" ||
    r.FromAccountID == r.ToAccountID || r.AmountCents <= 0 {
    return ErrInvalidTransfer
  }

  tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
  if err != nil { return fmt.Errorf("begin transfer: %w", err) }
  defer tx.Rollback()

  res, err := tx.ExecContext(ctx,
    "INSERT INTO transfers (transfer_id, from_account_id, to_account_id, amount_cents, status) "+
      "VALUES ($1, $2, $3, $4, 'pending') ON CONFLICT (transfer_id) DO NOTHING",
    r.ID, r.FromAccountID, r.ToAccountID, r.AmountCents)
  if err != nil { return fmt.Errorf("claim transfer ID: %w", err) }
  inserted, err := res.RowsAffected()
  if err != nil { return fmt.Errorf("inspect insert: %w", err) }

  if inserted == 0 {
    var from, to, state string
    var amount int64
    err := tx.QueryRowContext(ctx,
      "SELECT from_account_id::text, to_account_id::text, amount_cents, status "+
        "FROM transfers WHERE transfer_id = $1", r.ID,
    ).Scan(&from, &to, &amount, &state)
    if err != nil { return fmt.Errorf("read existing transfer: %w", err) }
    if from != r.FromAccountID || to != r.ToAccountID || amount != r.AmountCents {
      return ErrIdempotencyConflict
    }
    if state == "committed" { return nil }
    return errors.New("unexpected nonterminal transfer state")
  }

  ids := []string{r.FromAccountID, r.ToAccountID}
  sort.Strings(ids)
  balances := make(map[string]int64, 2)
  for _, id := range ids {
    b, err := lockBalance(ctx, tx, id)
    if err != nil { return err }
    balances[id] = b
  }
  if balances[r.FromAccountID] < r.AmountCents { return ErrInsufficientFunds }
  if balances[r.ToAccountID] > math.MaxInt64-r.AmountCents {
    return errors.New("destination balance overflow")
  }

  if _, err := tx.ExecContext(ctx,
    "UPDATE accounts SET balance_cents = balance_cents - $2 WHERE account_id = $1",
    r.FromAccountID, r.AmountCents); err != nil {
    return fmt.Errorf("debit source: %w", err)
  }
  if _, err := tx.ExecContext(ctx,
    "UPDATE accounts SET balance_cents = balance_cents + $2 WHERE account_id = $1",
    r.ToAccountID, r.AmountCents); err != nil {
    return fmt.Errorf("credit destination: %w", err)
  }
  res, err = tx.ExecContext(ctx,
    "UPDATE transfers SET status = 'committed' WHERE transfer_id = $1 AND status = 'pending'", r.ID)
  if err != nil { return fmt.Errorf("mark committed: %w", err) }
  n, err := res.RowsAffected()
  if err != nil || n != 1 { return fmt.Errorf("unexpected transfer state update: %w", err) }

  if err := tx.Commit(); err != nil { return &CommitUnknown{Cause: err} }
  return nil
}
~~~

The code assumes the schema above and a driver that supports PostgreSQL placeholders and row counts. It intentionally does not auto-retry. Production code must map database errors to stable API errors, classify retryable SQLSTATE codes, use a bounded request deadline, and instrument transaction duration, lock waits, retries, and unknown outcomes. Deferred rollback is harmless after commit. Cancellation racing with commit does not prove rollback.

### 8. Example Usage

~~~go
ctx, cancel := context.WithTimeout(request.Context(), 800*time.Millisecond)
defer cancel()

err := ledger.Transfer(ctx, db, ledger.TransferRequest{
  ID: idempotencyKey, FromAccountID: sourceID,
  ToAccountID: destinationID, AmountCents: 2500,
})
switch {
case err == nil:
  // Return success.
case errors.Is(err, ledger.ErrInsufficientFunds):
  // Return a business rejection.
case errors.Is(err, ledger.ErrIdempotencyConflict):
  // Reject key reuse with a different payload.
case errors.Is(err, ledger.ErrCommitOutcomeUnknown):
  // Retry using the same idempotency key.
default:
  // Map to a stable internal or retryable error.
}
~~~

The handler needs imports for context, errors, time, and the ledger package; request, database, and IDs are supplied by the HTTP layer. A successful transfer changes both balances once. A duplicate identical request returns success without another mutation. A payload mismatch is rejected. Insufficient funds rolls back. An uncertain commit is resolved with the same ID.

Use disposable PostgreSQL integration tests. Seed two accounts, run the same request twice, then assert balances changed once and one transfer record exists. Also test insufficient funds, payload mismatch, missing account, overflow, concurrent opposing transfers, and cancellation. Coordinate goroutines with barriers rather than sleeps and assert invariants after all finish. Mocks alone cannot validate real locking and isolation behavior.

### 9. Production Considerations

- Keep transactions short; never call remote services while holding locks.
- Transactions pin connections. Monitor pool wait time, long transactions, and idle-in-transaction sessions.
- Choose isolation from the invariant. Read Committed plus row locks can be sufficient for known rows; cross-row predicates may need other controls or Serializable.
- Retry only classified transient errors, as a whole transaction with a fresh transaction, bounded attempts, backoff, and jitter.
- A commit connection error can leave the result unknown. Retain idempotency records through the retry and reconciliation horizon.
- Hot rows serialize conflicting operations. More app instances do not remove correctness-required serialization.
- Stable lock order reduces but does not eliminate deadlocks.
- Propagate deadlines; cancellation can race with commit.
- Asynchronous replicas may lag after primary commit. Define read-after-write routing.
- Use a transactional outbox for event intent and make consumers idempotent.
- Validate data before adding constraints and plan online migrations for large tables.
- Authorize both accounts and avoid leaking sensitive account-existence details.
- Track transaction duration, commit latency, lock waits, deadlocks, retries, rollbacks, pool wait, and unknown outcomes. Avoid identifiers as metric labels.
- Validate backups by restoring them. Benchmark contention and failover, not just uncontended throughput.
- Transaction work consumes connection time, WAL, and index maintenance.

### 10. Tradeoffs

| Approach | Benefit | Cost or risk | Appropriate use |
|---|---|---|---|
| Read Committed plus row locks | Direct protection for known rows | Must identify all relevant rows | Row-scoped invariants |
| Repeatable Read | Stable snapshot in PostgreSQL | Some anomalies remain | Multi-read snapshot needs |
| Serializable | Serial-equivalent outcome | Whole-transaction retries | Complex cross-row predicates |
| Optimistic version | Low lock duration | Conflicts need handling | Low-contention updates |
| Pessimistic lock | Serializes conflicts | Contention and deadlocks | Exclusive mutation required |
| Application check only | Simple | Races can violate invariant | Not sufficient for critical rules |
| Transactional idempotency | Resolves duplicate/ambiguous requests | Retention and payload validation | Retried mutations |
| Blind retry with new ID | Simple-looking | Duplicates committed work | Never use for ambiguous commit |
| Local transaction plus outbox | Atomic state and event intent | Async delivery and reconciliation | Database changes must emit events |
| Saga | Coordinates separate owners | Intermediate states and compensation | Cross-service workflows |

Isolation strength cannot replace invariant modeling. Use the least complex mechanism that protects the required state transition.

### 11. Connection to Previous Lessons

Day 7, SD-007, defined schema, keys, constraints, and access patterns. SD-008 explains how concurrent operations preserve those invariants. Day 6, SD-006, established authoritative ownership; a local transaction protects only its transactional authority. Day 5, SD-005, established idempotency and API error semantics, essential when commit outcomes are uncertain. Day 3, SD-003, supplies workload estimates for contention and pool sizing.

The Day 7 lesson exists on main, so its prerequisite is satisfied despite stale progress metadata. Previous exercises remain pending because no learner submissions have been reviewed; they do not block this lesson.

### 12. What Comes Next

**SD-009: Index Structures and Query Planning**, Level 2. With schema and transaction behavior established, the next concept explains B-tree indexes, selectivity, composite-key order, query plans, and write amplification. SD-009 depends on SD-008, covered here.

SD-011, SD-012, SD-013, and SD-015 are also eligible after status reconciliation, but SD-009 is next in the storage progression. It is not taught here.

### 13. Mini Exercise

Two concurrent requests reserve one SKU. Available quantity is 7 and each request wants 5 units. Describe two correct strategies, a conditional atomic update and a row lock. Explain what each reads, which statement enforces the invariant, what the losing request observes, and what happens if commit acknowledgement is lost.

**Acceptance criteria:** No execution commits a negative quantity; isolation and locking assumptions are explicit; concurrent behavior is described as an interleaving; retries use a stable operation ID; and rollback is distinguished from an unknown commit result. Do not provide the complete solution here.

### Key Takeaways

- A transaction defines which database changes commit together, not which remote side effects happen atomically.
- Isolation levels permit different concurrent histories; understand the engine's guarantees.
- Constraints protect invariants against races and alternate writers.
- Row locks serialize conflicting work; stable lock order reduces deadlock risk.
- Retry known transient conflicts as a whole transaction with bounded attempts and a fresh transaction.
- A failed commit acknowledgement is an unknown outcome, not proof of rollback.
- Durable idempotency records resolve duplicate and ambiguous requests.
- Cross-service atomicity requires a broader protocol, such as an outbox or saga.

### Concept Map

~~~mermaid
flowchart TD
  A[SD-007: Data modeling] --> B[SD-008: Relational transactions]
  B --> C[SD-009: Indexes and query planning]
  B --> D[SD-010: Query optimization and pooling]
  B --> E[SD-019: Replication]
  B --> F[SD-023: Distributed transactions and sagas]
  G[SD-005: API contracts] --> B
  B --> H[SD-031: Transactional outbox]
  E --> I[SD-020: Consistency models]
~~~

### Exercise

Design and implement a transaction-safe inventory reservation service in PostgreSQL and Go.

**Deliverables:** schema with constraints and idempotency uniqueness; transaction implementation that prevents overselling; isolation/locking rationale; stable idempotency behavior; PostgreSQL integration tests for duplicates, payload mismatch, insufficient inventory, rollback, cancellation, and concurrency; bounded retries for classified transient errors; metrics for transaction duration, lock waits, retries, and unknown outcomes; a load-test plan varying SKU hotness; and failure analysis for crashes, lost commit acknowledgement, failover, and event publication.

**Acceptance criteria:** constraints enforce invariants; concurrent requests cannot oversell; identical retries apply at most once within the retention window; mismatched payloads are rejected; retries are bounded and correctly classified; tests run against PostgreSQL; performance claims are measured; and setup is reproducible. Pending until submitted and reviewed.

### Progress

- **Day:** 8, based on the Day 7 lesson and absence of a Day 8 lesson on main before this run.
- **Level:** Level 2, Storage and request-path mechanics.
- **Current concept:** SD-008, Relational databases and transaction semantics.
- **Lesson path:** system-design/lessons/day-08-relational-databases-and-transaction-semantics.md
- **Prerequisites:** SD-007 covered by the existing Day 7 lesson.
- **Mini exercise:** Pending learner submission.
- **Substantial exercise:** Pending learner submission.
- **Concepts covered after persistence:** SD-001 through SD-008.
- **Next concept:** SD-009, Index structures and query planning. SD-011, SD-012, SD-013, and SD-015 are also eligible.
- **Repository status:** Verify lesson, curriculum, and progress by reading them back from main.

### Most Important Mental Model

**A transaction is a database-enforced state transition: define the invariant, protect every concurrent path that can violate it, and resolve unacknowledged commits idempotently.**
