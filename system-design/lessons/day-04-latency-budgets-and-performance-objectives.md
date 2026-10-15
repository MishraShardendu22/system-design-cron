# DAY 4: Latency Budgets and Performance Objectives
**Level:** 1, Foundations. **Concept:** SD-004. **Prerequisites:** SD-002 and SD-003 covered. **Status:** Delivered; exercises pending.
### 1. Why It Exists
Average throughput hides slow requests. Queueing, pool waits, serial calls, and slow dependencies degrade p99 while p50 stays healthy. Independent timeouts may pass while the request misses its objective. Retries that reset timeouts amplify overload.
### 2. Core Idea
A **latency objective** is a measurable target for a request population. A **latency budget** allocates time across the critical path. A **timeout** bounds a local wait; a **deadline** is the absolute end of useful work.
Account for ingress, queueing, application, dependencies, serialization, and egress. Specify population, boundary, workload, target, window, stage limits, and response reserve. A 200 ms target reserving 15 ms for response leaves 185 ms. Component p99 values do not mechanically sum; measure end to end.
### 3. Mental Model
The deadline is a finite time account. Available time = deadline - now - response reserve. A stage gets the smaller of its allocation and available time. If none remains, do not start optional work.
### 4. Real-World Example
Hypothetical API: p99 reads <= 200 ms at 10,000 requests/second. With 15 ms reserved for response, stages must total <= 185 ms.
**Documented fact:** Go context supports deadlines/cancellation, https://pkg.go.dev/context. **Industry practice:** trace stages, propagate deadlines, bound queues, and load-test. **Inference:** a 90 ms query violates a 70 ms cap; inspect pool wait, execution, locks, and I/O before scaling.
### 5. Low-Level Design (LLD)
A plan has total duration, response reserve, and named stage limits. Validate allocations at startup. Establish one request deadline; derive child contexts from stage caps and remaining time. Never extend the parent deadline. Call cancel functions. Cancellation is not rollback. Keep the plan immutable.
### 6. High-Level Design (HLD)
Ingress establishes the deadline. The handler propagates context through queueing, database, and policy stages, then reserves time for the response. Trace end-to-end and stage latency. Distinguish client cancellation, stage timeout, and overall expiry. Retries share the deadline; budgets do not replace admission control.
### 7. Realistic Go Code
\`\`\`go
func StageContext(parent context.Context, limit, reserve time.Duration) (context.Context, context.CancelFunc, error) {
 if limit <= 0 || reserve < 0 { return nil,nil,errors.New("invalid budget") }
 d, ok := parent.Deadline()
 if !ok { return nil,nil,errors.New("deadline required") }
 left := time.Until(d)-reserve
 if left <= 0 { return nil,nil,errors.New("budget exhausted") }
 if left < limit { limit = left }
 ctx, cancel := context.WithTimeout(parent, limit)
 return ctx, cancel, nil
}
\`\`\`
Import context, errors, and time. Establish the request deadline first; pass the child context to I/O and call cancel. Verify driver cancellation behavior.
### 8. Example Usage
Pass the child context to QueryRowContext or NewRequestWithContext. Test exhausted deadlines and cancellation with go test ./... and a fake slow dependency.
### 9. Production Considerations
Measure aligned percentiles; include queue/pool wait; charge retries to the same deadline; use low-cardinality metrics; validate config; define fallbacks.
### 10. Tradeoffs
One timeout is simple but offers little stage control. Independent timeouts can exceed the total budget. Shared deadlines with stage caps preserve the total bound but require propagation. Measured allocations need telemetry; retries/hedges add load.
### 11. Connection to Previous Lessons
Days 1-3 covered tradeoffs, requirements, and workload. SD-004 depends on covered SD-002/003. Requirements define the target; workload defines conditions; budgets constrain the path.
### 12. What Comes Next
**SD-005: API Design, Contracts, and Versioning**, Level 1. SD-002 is covered. API semantics define behavior, idempotency, pagination, errors, and compatibility. Not taught here.
### 13. Mini Exercise
For a 250 ms p99 endpoint with 20 ms response reserve and stage limits 15/100/90 ms, check feasibility, revise with 10 ms margin, and calculate policy time left after 120 ms.
**Acceptance criteria:** Show arithmetic, preserve reserve, distinguish timeout/deadline, and specify an end-to-end test.
### Key Takeaways
- Objectives need population, boundary, workload, target, and window.
- Every stage spends from one deadline.
- Component percentiles do not mechanically sum.
- Queueing and retries consume time.
- End-to-end measurement validates budgets.
### Concept Map
\`\`\`mermaid
flowchart TD
 A[SD-002 Requirements] --> C[SD-004 Latency budgets]
 B[SD-003 Capacity] --> C
 C --> D[SD-005 API design]
 C --> E[SD-041 Observability]
\`\`\`
### Exercise
Design a Go policy with stage validation, one deadline, DB/HTTP propagation, bounded metrics, slow-dependency and load tests, and retry rules.
**Acceptance criteria:** Tests pass; child contexts are canceled; deadlines never extend; invalid plans fail before serving; measured latency is compared with target. Pending until review.
### Progress
- **Day/level:** 4, Level 1. **Concept:** SD-004.
- **Path:** system-design/lessons/day-04-latency-budgets-and-performance-objectives.md
- **Prerequisites:** SD-002 and SD-003 covered.
- **Exercises:** Both pending. **Covered:** SD-001 through SD-004.
- **Next:** SD-005, API design, contracts, and versioning.
- **Status:** Verify lesson and trackers after persistence.
### Most Important Mental Model
**Every synchronous stage spends from one finite request deadline; no downstream timeout creates more time.**
