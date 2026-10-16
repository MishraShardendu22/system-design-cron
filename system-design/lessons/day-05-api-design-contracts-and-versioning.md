# DAY 5: API Design, Contracts, and Versioning

**Progression level:** Level 1. **Concept:** SD-005. **Prerequisite:** SD-002 covered.

### 1. Why It Exists
An API is a compatibility boundary. Ambiguous retry, error, pagination, and evolution semantics cause duplicate writes and client breakage.

### 2. Core Idea
Define request/response schemas, stable errors, idempotency, pagination, and version policy.

### 3. Mental Model
Treat the API as a protocol between independently deployed state machines.

### 4. Real-World Example
A document API must recover safely when a create request commits but its response is lost.

### 5. Low-Level Design (LLD)
Use a normalized request fingerprint and an atomic idempotency record.

### 6. High-Level Design (HLD)
Client -> API validation -> durable store -> stable response.

### 7. Realistic Go Code
```go
package main
import "net/http"
func main() { _ = http.ListenAndServe(":8080", http.NotFoundHandler()) }
```

### 8. Example Usage
`go run main.go` starts the example HTTP server.

### 9. Production Considerations
Use shared durable state, bounded inputs, authorization, observability, and contract tests.

### 10. Tradeoffs
Path versioning is explicit but increases version support cost. Cursor pagination scales better than deep offsets but has more complex semantics.

### 11. Connection to Previous Lessons
Days 1-4 covered fundamentals, requirements, capacity, and latency. SD-005 depends on SD-002.

### 12. What Comes Next
SD-006, Domain Boundaries and Data Ownership.

### 13. Mini Exercise
Specify idempotency behavior for a payment API. Acceptance: document same-key replay, conflicting payloads, and ambiguous provider outcomes.

### Key Takeaways
Contracts specify observable behavior. Idempotency requires durable atomic state. Compatibility includes semantics.

### Concept Map
SD-002 -> SD-005 -> SD-006; SD-005 also enables SD-015.

### Exercise
Create an OpenAPI spec, compatibility policy, and tests for retries, errors, and pagination. Acceptance: implementation and specification agree; pending review.

### Progress
Day 5, Level 1, SD-005. Prerequisite SD-002 covered. Exercises pending. Next SD-006. Persistence verification pending.

### Most Important Mental Model
A good API contract makes retries, failures, and evolution predictable.
