# System Design Curriculum

A dependency-ordered, persistent curriculum for developing production-grade system-design judgment, from foundational architectural reasoning through distributed systems and end-to-end production architecture.

## Learning objectives

- Translate product requirements into explicit functional, scale, latency, durability, and availability constraints.
- Reason about data ownership, APIs, storage, caching, replication, partitioning, and consistency.
- Design for concurrency, partial failure, overload, recovery, and operational visibility.
- Explain architectural alternatives with quantified assumptions and explicit tradeoffs.
- Validate designs through realistic Go implementations, tests, capacity estimates, and failure analysis.

## Progression levels

1. **Foundations:** architectural reasoning, requirements, estimation, interfaces, and core data models.
2. **Data and request paths:** database behavior, indexes, caching, proxies, and load balancing.
3. **Scale and distributed state:** replication, partitioning, consistency, coordination, and consensus.
4. **Asynchronous systems:** queues, event-driven architecture, delivery semantics, and stream processing.
5. **Production reliability:** resilience, observability, SLOs, capacity, security, tenancy, and infrastructure.
6. **Architecture synthesis:** end-to-end designs, migrations, and advanced tradeoffs.

## Prerequisites

Assume working knowledge of backend engineering, networking, databases, concurrency, Linux, and Go. Each lesson identifies prerequisites explicitly. A concept is eligible only when all prerequisite concepts have been covered or the curriculum documents a justified exception.

## Daily workflow

Each scheduled execution inspects `main`, reads this file, `curriculum.md`, `progress.md`, and relevant existing lessons, then selects exactly one uncovered concept whose dependencies are satisfied. It writes one complete lesson, updates curriculum status and progress, validates consistency, and commits curriculum-only changes to `main` when repository permissions allow. Day numbers follow lesson history, not calendar dates. Lesson delivery does not imply exercise completion.

## File structure

- `README.md`: purpose, workflow, and conventions.
- `curriculum.md`: concept identifiers, dependency graph, objectives, levels, and statuses.
- `progress.md`: verified learning state, exercise status, gaps, and next eligible concept.
- `lessons/day-NN-<topic>.md`: one complete lesson per execution.

## Lesson conventions

Lessons use zero-padded day numbers and a kebab-case topic. Every lesson contains the prescribed sections in order, a realistic Go implementation, example usage, production considerations, tradeoffs, exercises, a concept map, and a verified progress report. External factual claims are cited. Documented facts, established industry practice, and engineering inference are distinguished.

## Progress-tracking rules

- The repository is the source of truth; inspect actual files and commit history before each lesson.
- Curriculum statuses are `planned`, `eligible`, `in_progress`, or `covered`; covered means a lesson exists, not that its exercises are complete.
- Progress records lesson delivery separately from mini-exercise and substantial-exercise completion.
- Do not skip dependencies or repeat concepts already covered. Pending exercises do not block unrelated next concepts unless the missing understanding is a prerequisite.
- Modify only curriculum-related files. Preserve unrelated content and concurrent changes.
