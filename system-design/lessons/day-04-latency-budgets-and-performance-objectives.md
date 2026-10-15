# DAY 4: Latency Budgets and Performance Objectives

**Progression level:** Level 1, Foundations and architectural reasoning  
**Concept ID:** SD-004  
**Prerequisites:** SD-002 and SD-003, both covered  
**Status:** Lesson delivered. Exercises pending.

### 1. Why It Exists

Average throughput does not describe the experience of a slow request. A request may queue behind saturated workers, wait for a database connection, execute serial network calls, or encounter a slow dependency. The median can remain healthy while p99 latency becomes unacceptable. Day 3 capacity estimates quantify offered work, but do not define how much time each stage may consume.

A target such as p99 <= 200 ms is useful only with a defined request population, measurement boundary, workload, and window. Without a budget, components can each consume time independently until no time remains to serialize the response. Without deadline propagation, downstream work may continue after the caller has timed out, wasting capacity. Retries that reset a full timeout can amplify an already overloaded dependency.

A latency budget turns an end-to-end objective into explicit constraints across the critical path. It reveals infeasible designs early and prevents local health metrics from hiding poor user-visible latency.

### 2. Core Idea

A **latency objective** is an observable target for a defined request population. A **latency budget** allocates time to request stages. A **timeout** bounds how long a caller waits for an operation. A **deadline** is the absolute time by which the work is no longer useful. These are related but not interchangeable.

A simple request-time accounting model is:

**End-to-end time = ingress + queueing + application + dependencies + serialization + egress**

Some stages overlap or are conditional, so this is an accounting model for a specified path, not a claim that component percentiles can be added mechanically.

A practical budget specifies:
- **Population and boundary:** which requests count, and where timing starts and ends.
- **Workload:** arrival rate, operation mix, payload distribution, cache state, and failure conditions.
- **Target:** for example, p99 <= 200 ms measured over rolling five-minute windows.
- **Stage allocations:** bounded time for queueing, application work, and synchronous dependencies.
- **Response reserve:** time protected for serialization, framework overhead, and egress.
- **Deadline propagation:** how remaining request time constrains downstream work.
- **Tail-risk allocation:** acceptable probability of stage-threshold exceedance.

If a 200 ms target reserves 15 ms for response work, at most 185 ms remains for allocated stages. A plan whose allocations exceed 185 ms is invalid before deployment. Allocations are limits to validate, not guarantees that a stage will always finish within them.

**Percentiles:** In general, end-to-end p99 is not the sum of component p99 values. Slow events may be correlated, and component request populations may differ. A conservative tail-risk allocation can use the union bound: if four required stages each have at most 0.25% probability of exceeding a defined threshold for the same request population, the probability that any stage exceeds its threshold is at most 1%. The stage thresholds must sum within the end-to-end budget, and the actual end-to-end distribution must still be measured.

**Queueing matters.** A fast function behind an overloaded queue can be slow to its caller. At high utilization, small changes in arrival rate or service-time variance can disproportionately increase tail latency. Little's Law from Day 3 helps estimate in-flight work, but does not predict p99 without assumptions about arrivals, service-time distributions, and queueing discipline.

Each downstream operation should receive a timeout bounded by its stage allocation and the remaining request deadline minus response reserve. Retries consume the same deadline, including backoff. Cancellation can stop supported work, but cannot undo a side effect that has already committed.

### 3. Mental Model

Treat the request deadline as a **finite time account**. Every synchronous stage spends from the same account. A stage has a local spending limit, but cannot create more total time. Reserve time for response work.

For deadline D, current time now, and response reserve R:

**available = D - now - R**

The dependency timeout is the smaller of its configured allocation and available time. If available <= 0, do not start another dependency call. Stop optional work and follow the documented timeout or degraded-response behavior.

At each boundary ask: how much time remains, which work is mandatory, and what evidence shows the objective is met?

This model is not a substitute for measurements. Equal allocations are rarely optimal, and deadlines do not guarantee that remote systems stop after cancellation. Stage budgets are hypotheses to validate through tracing and representative load tests.


### 4. Real-World Example

A hypothetical document API has a p99 server-side read objective of 200 ms at 10,000 requests/second. These values are illustrative, not claims about an existing system. A candidate request path must reserve time for response serialization and egress. Stage allocations are hypotheses to validate with end-to-end traces and load tests, not proof that the overall percentile meets its target.


Additional verification note: stage limits are hypotheses, and end-to-end measurements remain authoritative.
