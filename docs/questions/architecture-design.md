# Software Architecture & Design Questions

> 25 architecture trade-off discussions for Staff/Principal level interviews.
> Focus: When to use what, real-world trade-offs, and what separates SDE 3 from SDE 4 answers.

---

### A1: Monolith vs Microservices When Each Wins

**Context:** Startup has a 50K LOC monolith serving 1M users. Considering microservices.

**Expected Discussion:**

| Factor | Monolith Wins | Microservices Win |
|--------|--------------|-------------------|
| Team size | < 10 devs | 5+ teams (50+ devs) |
| Deployment | Simple, one artifact | Independent deployment per team |
| Latency | In-process calls (ns) | Network calls (ms) |
| Debugging | Single process, easy | Distributed tracing needed |
| Consistency | ACID transactions | Eventual consistency, sagas |
| Conway's Law | Single team | Team per domain |

**SDE 3 says:** "Microservices for better scaling and deployment independence."
**SDE 4 says:** "Start monolith (modulith), split when team boundaries or scaling needs force it. Premature decomposition is worse than a monolith. Look for: independent deployment need, different scaling profiles, team cognitive load."

**Follow-up:** What's a modular monolith? → Monolith with enforced module boundaries (internal APIs). Can split later without rewrite.

---

### A2: Event-Driven vs Request-Response

**Context:** E-commerce order flow should downstream services (inventory, shipping, email) be called synchronously or via events?

**Expected Discussion:**

| Aspect | Request-Response | Event-Driven |
|--------|-----------------|--------------|
| Coupling | Tight (caller knows callee) | Loose (producer doesn't know consumers) |
| Latency | Synchronous, predictable | Async, eventual |
| Failure handling | Immediate error | Dead letter queues, retry |
| Debugging | Easy (call stack) | Hard (event flow) |
| Scalability | Limited by slowest service | Independent scaling |

**When event-driven:** Fire-and-forget notifications, fan-out to multiple consumers, different SLAs across downstream services, spike absorption.

**When request-response:** Need immediate confirmation, synchronous business logic (e.g., payment authorization before order confirm).

**SDE 4 insight:** Hybrid is almost always the answer. Synchronous for the critical path (payment), events for everything else (email, analytics, recommendations).

---

### A3: CQRS + Event Sourcing When Worth the Complexity

**Context:** Financial system with audit requirements and complex read patterns.

**Expected Discussion:**

**Worth it when:**
- Complete audit trail is a business requirement (finance, healthcare).
- Read and write models are radically different (normalized writes, denormalized reads).
- Temporal queries needed ("What was the account balance at 3pm yesterday?").
- High read:write ratio (materialize multiple read views from same events).

**Not worth it when:**
- Simple CRUD with no complex querying.
- Team is small and unfamiliar with the pattern.
- Eventual consistency is unacceptable for the domain.

**Key trade-offs:** Complexity (debugging, projections), eventual consistency between write and read sides, storage growth (append-only), snapshot management.

**SDE 4 insight:** Event sourcing without CQRS is rarely useful. CQRS without event sourcing is common and simpler (just separate read/write DBs).

---

### A4: Database per Service vs Shared Database

**Context:** 5 microservices currently sharing one PostgreSQL instance.

| Approach | Pros | Cons |
|----------|------|------|
| Shared DB | Joins, transactions, simple | Schema coupling, deployment coupling, single point of failure |
| DB per service | Independent schemas, scale independently, polyglot persistence | No joins, distributed transactions, data duplication |

**Migration path:** Start with schema-per-service (same instance), then separate instances when needed.

**SDE 4 insight:** The real question is "what data is truly owned by one service?" If two services always need the same data in transactions, maybe they're one service.

---

### A5: Synchronous vs Asynchronous Communication

**Context:** Service A needs data from Service B to complete a request.

**Decision framework:**
- Sync (HTTP/gRPC): Need response now, simple retry, request-response fits naturally.
- Async (message queue): Can tolerate delay, fire-and-forget, spike buffering, fan-out.
- Async with reply (correlation ID): Need response but can wait, decouple timing.

**SDE 4 insight:** Think about failure modes. Sync: if B is down, A is down (cascade). Async: if B is down, queue buffers (graceful degradation). Design for the failure case, not the happy path.

---

### A6: API Versioning Strategies

**Context:** Public API serving 1000+ clients. Need breaking change.

| Strategy | URL | Header | Content Negotiation |
|----------|-----|--------|-------------------|
| Example | `/v1/users` | `Api-Version: 2` | `Accept: application/vnd.api.v2+json` |
| Pros | Simple, discoverable | Clean URLs | HTTP-correct |
| Cons | URL pollution | Easy to miss | Complex |
| Caching | Easy (URL-based) | Vary header needed | Complex |

**SDE 4 approach:** Prefer backward-compatible evolution over versioning. Add fields (don't remove), use optional parameters, deprecate gradually. Version only when truly breaking.

**Sunset policy:** Announce deprecation 6 months ahead. Monitor old version usage. Kill when <1% traffic.

---

### A7: Circuit Breaker vs Retry vs Timeout

**Context:** Service A calls Service B which occasionally fails or is slow.

**Layered resilience:**
```
Request → Timeout (2s) → Retry (3x, exponential backoff) → Circuit Breaker (5 failures → open)
```

| Pattern | Purpose | When |
|---------|---------|------|
| Timeout | Bound wait time | Always prevent resource exhaustion |
| Retry | Handle transient failures | Idempotent operations only |
| Circuit breaker | Prevent cascade, allow recovery | Dependency with known failure modes |
| Bulkhead | Isolate failures | Multiple downstream dependencies |
| Fallback | Graceful degradation | Can serve stale data or default |

**SDE 4 insight:** Retries amplify load on failing services. Always combine with backoff + jitter. Circuit breaker is useless without fallback behavior defined.

---

### A8: Consistent Hashing Why and When

**Context:** Distributing cache keys across N servers. Adding/removing servers.

**Problem with modular hashing:** `hash(key) % N` changing N redistributes almost all keys.

**Consistent hashing:** Only K/N keys move when adding a node (K = total keys, N = nodes).

**Virtual nodes:** Each physical node maps to 100-200 points on the ring. Ensures even distribution and handles heterogeneous hardware.

**Used in:** DynamoDB, Cassandra, Redis Cluster, CDN routing, load balancing.

**SDE 4 insight:** Jump consistent hash is simpler and more uniform for the case where nodes are numbered sequentially (no arbitrary removal). Rendezvous hashing is better when you need multi-probe (replication).

---

### A9: CAP Theorem Real-World Examples

**Context:** Designing a distributed system which guarantee to sacrifice?

| System | Choice | Sacrifices | Reasoning |
|--------|--------|-----------|-----------|
| Bank ledger | CP | Availability during partition | Incorrect balance is unacceptable |
| DNS | AP | Consistency (stale records OK) | Must always respond |
| Social media feed | AP | Consistency (eventual OK) | Stale feed > no feed |
| Payment system | CP | Availability (reject if unsure) | Double-charge is worse than downtime |
| Shopping cart | AP | Consistency (merge on reunion) | Can always add to cart |

**SDE 4 insight:** CAP is about behavior during network partitions. In normal operation, you have all three. The real framework is PACELC: during Partition, choose A or C; Else, choose Latency or Consistency.

---

### A10: Saga Pattern vs 2PC for Distributed Transactions

**Context:** Order service needs to: reserve inventory + charge payment + schedule shipping.

| Approach | 2PC (Two-Phase Commit) | Saga |
|----------|----------------------|------|
| Consistency | Strong (ACID across services) | Eventual (compensating transactions) |
| Availability | Low (coordinator is SPOF) | High (no central coordinator) |
| Performance | Slow (lock held across network) | Fast (no distributed locks) |
| Complexity | Protocol complexity | Compensation logic |
| Recovery | Coordinator log | Idempotent compensations |

**Saga types:**
- Choreography: Services emit events, others react. Simple but hard to track.
- Orchestration: Central coordinator directs the flow. Easier to debug.

**SDE 4 insight:** 2PC across microservices is an anti-pattern (defeats the purpose). Use sagas with semantic locks (soft-state reservations) and timeouts for cleanup.

---

### A11: Service Mesh When It's Overkill

**Worth it:** 50+ microservices, polyglot (mixed languages), need mTLS everywhere, complex traffic management (canary, fault injection), team can't modify apps.

**Overkill:** <10 services, single language (can use library-based approach), team is small, adds operational complexity that outweighs benefits.

**What it provides:** mTLS, observability (automatic metrics/traces), traffic management, retries/timeouts, access control all without app code changes.

**Alternatives:** Shared libraries (e.g., Spring Cloud), API gateway for north-south traffic only.

---

### A12: Multi-Tenancy Shared vs Isolated Infrastructure

| Model | Shared everything | Shared app, separate DB | Fully isolated |
|-------|-------------------|------------------------|----------------|
| Cost | Lowest | Medium | Highest |
| Isolation | Noisy neighbor risk | Data isolated | Full isolation |
| Complexity | Tenant-aware code | Schema/connection management | Deployment per tenant |
| Compliance | Hard (data mingling) | Good (separate data) | Easy (physical separation) |
| Use case | SaaS, small tenants | Enterprise SaaS | Enterprise, regulated |

**SDE 4 insight:** Start shared, offer isolation as premium tier. Row-level security + tenant_id column is the most cost-effective for most SaaS.

---

### A13: Data Partitioning Strategies

| Strategy | Range | Hash | List |
|----------|-------|------|------|
| Distribution | Uneven (hot ranges) | Even | Grouped by attribute |
| Range queries | Efficient | Requires scatter-gather | Within partition only |
| Rebalancing | Split hot ranges | Consistent hashing | Manual |
| Use case | Time-series data | User data, general KV | Geographic/categorical |

**SDE 4 insight:** Composite partitioning (hash + range) solves most issues. E.g., partition by user_id (hash) then sort by timestamp within partition.

---

### A14: Cache Invalidation Strategies

| Strategy | TTL-based | Write-through | Write-behind | Event-driven |
|----------|-----------|--------------|--------------|--------------|
| Consistency | Stale for TTL | Strong | Eventual | Near real-time |
| Complexity | Low | Medium | High | Medium |
| Write perf | Fast (no cache update) | Slower (sync write) | Fast (async) | Fast |
| Read perf | Fast (cached) | Fast | Fast | Fast |

**Patterns:**
- Cache-aside: App manages cache. Read miss → load from DB → populate cache.
- Read-through: Cache itself loads from DB on miss.
- Write-through: Write to cache + DB synchronously.
- Write-behind: Write to cache, async flush to DB.

**SDE 4 insight:** "There are only two hard things: cache invalidation and naming things." In practice: TTL + event-driven invalidation covers 95% of cases. Don't cache data that changes frequently AND must be consistent.

---

### A15: Idempotency in Distributed Systems

**Context:** Payment service receives duplicate webhook. How to prevent double-charging?

**Implementation:**
```
1. Client sends request with idempotency key (UUID)
2. Server checks: key exists in idempotency store?
   - Yes: return cached response (don't re-execute)
   - No: execute, store result with key, return
3. Key expires after 24-48 hours
```

**Challenges:** Concurrent duplicate requests (use DB unique constraint on key). Partial failures (store key BEFORE execution to prevent re-execution, but handle rollback).

**SDE 4 insight:** Make ALL write operations idempotent at the API layer. Use natural idempotency keys where possible (order_id, not random UUID). Combine with at-least-once delivery for reliability.

---

### A16: Backwards-Compatible API Evolution

**Rules for non-breaking changes:**
1. Add new optional fields (never remove or rename)
2. Add new endpoints (don't change existing behavior)
3. Widen input types (accept string where int was required)
4. Narrow output types only if clearly documented

**Breaking change mitigation:**
- Feature flags: New behavior behind flag, old clients unaffected.
- Expand-and-contract: Add new field → migrate clients → remove old field.
- Deprecation headers: `Sunset: Sat, 31 Dec 2025 23:59:59 GMT`

---

### A17: Zero-Downtime Database Migrations

**Pattern: Expand and Contract**
```
Phase 1 (Expand): Add new column, nullable. Deploy app that writes to BOTH old and new.
Phase 2 (Migrate): Backfill new column from old data.
Phase 3 (Contract): Deploy app that reads from new column only. Drop old column.
```

**Rules:**
- Never rename columns in production.
- Never add NOT NULL without default in a single step.
- Never drop columns that running code reads.
- Use online DDL tools (pt-osc, gh-ost) for large tables.
- Each migration step must be independently deployable and rollback-safe.

**SDE 4 insight:** The discipline is: every schema change must be compatible with BOTH the current and next version of the application running simultaneously.

---

### A18: Eventual Consistency Handling in UX

**Context:** User creates a post but doesn't see it immediately in their feed (replication lag).

**Strategies:**
- **Read-your-writes:** After write, read from primary (or include local state in UI).
- **Optimistic UI:** Show the action as successful immediately, reconcile later.
- **Causal consistency:** Ensure a user sees their own updates in order.
- **Monotonic reads:** Never show older state after showing newer state.

**SDE 4 insight:** Most users don't notice 1-2s of eventual consistency. The cases that matter: financial (show pending state), social (read-your-writes), collaborative (conflict resolution UI).

---

### A19: Rate Limiting Algorithms and Trade-offs

| Algorithm | Burst | Precision | Memory | Complexity |
|-----------|-------|-----------|--------|-----------|
| Token Bucket | Allows bursts up to bucket size | Good | O(1) per key | Low |
| Leaky Bucket | Smooth output, no bursts | Good | O(1) per key | Low |
| Fixed Window | Boundary burst (2× at window edge) | Low | O(1) per key | Lowest |
| Sliding Window Log | No burst artifacts | Exact | O(N) per key | Medium |
| Sliding Window Counter | Minimal burst artifacts | Approximate | O(1) per key | Low |

**SDE 4 insight:** Token bucket for most APIs (allows healthy bursts). Sliding window counter for billing (accurate without memory cost). Leaky bucket for outbound calls to rate-limited dependencies.

---

### A20: Load Balancing L4 vs L7, Algorithms

| | L4 (Transport) | L7 (Application) |
|---|------|------|
| Layer | TCP/UDP | HTTP/gRPC |
| Speed | Faster (no payload inspection) | Slower (parse headers/body) |
| Features | Connection-based routing | URL/header/cookie routing, TLS termination |
| Use case | Raw throughput, database | API routing, canary, A/B testing |

**Algorithms:**
- Round Robin: Simple, ignores server capacity.
- Weighted Round Robin: Accounts for heterogeneous servers.
- Least Connections: Best for variable request duration.
- Consistent Hashing: Session affinity without sticky sessions.
- Random with Two Choices (P2C): O(1) and near-optimal. Pick 2 random servers, choose less loaded.

**SDE 4 insight:** P2C (power of two choices) is the sweet spot for most distributed systems. Netflix and Envoy use it.

---

### A21: Message Ordering Guarantees

| Level | Guarantee | Implementation | Cost |
|-------|-----------|---------------|------|
| No ordering | Messages arrive in any order | Multiple partitions, any consumer | Cheapest |
| Partition ordering | Ordered within partition key | Kafka partition, SQS FIFO group | Medium |
| Total ordering | Global order across all messages | Single partition / Raft consensus | Expensive |
| Causal ordering | Causally related messages in order | Vector clocks / Lamport timestamps | Complex |

**SDE 4 insight:** Total ordering is almost never needed at scale. Design for partition ordering (e.g., all events for user_123 are ordered). Causal ordering is needed for collaborative systems.

---

### A22: Dead Letter Queue Design

**Purpose:** Store messages that repeatedly fail processing for later investigation.

**Architecture:**
```
Main Queue → Consumer → Process
                 ↓ (failure after N retries)
             DLQ (separate queue)
                 ↓
          Alert + Dashboard → Manual inspection → Replay
```

**Key decisions:**
- Retry count before DLQ: 3-5 with exponential backoff.
- DLQ retention: 14 days (enough for investigation).
- Metadata: Include original queue, error message, attempt count, timestamp.
- Replay: Tool to re-queue DLQ messages to original queue after fix.
- Alerting: Alert when DLQ size grows (indicates systemic issue).

---

### A23: Health Checks Liveness vs Readiness vs Startup

| Probe | Purpose | Failure Action | Example |
|-------|---------|---------------|---------|
| Startup | Is the app initialized? | Keep waiting (don't restart yet) | DB migration running |
| Liveness | Is the process alive? | Kill and restart | Deadlock, OOM |
| Readiness | Can it serve traffic? | Remove from load balancer | DB connection lost, warming cache |

**SDE 4 insight:** Common mistake: putting dependency checks in liveness probes. If DB is down and liveness fails, K8s restarts ALL pods simultaneously → cascade failure. Dependency checks go in readiness (stop traffic) not liveness (restart).

---

### A24: Graceful Degradation Under Load

**Strategies (ordered by severity):**
1. **Shed load:** Return 429/503 for excess requests (rate limit).
2. **Reduce quality:** Serve cached/stale data, skip personalization.
3. **Disable non-critical features:** Turn off recommendations, analytics.
4. **Circuit break:** Stop calling failing dependencies, return defaults.
5. **Static fallback:** Serve pre-generated static pages.

**Implementation pattern:**
```
Load Level:
  Normal (< 70% capacity): Full feature set
  Elevated (70-85%): Disable expensive features (ML scoring, real-time recommendations)
  Critical (85-95%): Serve cached responses, skip DB writes for non-essential
  Emergency (> 95%): Static responses, alert on-call
```

**SDE 4 insight:** Graceful degradation must be tested regularly (chaos engineering). If it only activates in production emergencies, it's probably broken. Run "load shedding drills" quarterly.

---

### A25: Technical Debt Quantification and Paydown Strategy

**Quantification framework:**
```
Debt Impact Score = Frequency of Pain × Severity × Number of People Affected

Categories:
  - Code debt: Complex functions, missing tests, tight coupling
  - Architecture debt: Wrong abstraction, scaling limits
  - Infrastructure debt: Manual processes, outdated dependencies
  - Knowledge debt: Missing docs, tribal knowledge
```

**Paydown strategies:**

| Strategy | When | Example |
|----------|------|---------|
| Boy Scout Rule | Always | Leave code better than you found it (small improvements) |
| 20% time | Sustained | Reserve 20% of sprint for tech debt |
| Focused sprint | Quarterly | Dedicate full sprint to infrastructure upgrades |
| Strangler fig | Large rewrites | Gradually replace old system with new |
| Sunset deadline | End of life | Set date, migrate clients, decommission |

**SDE 3 says:** "We should refactor this module."
**SDE 4 says:** "This debt costs us 2 days/sprint in incident response and 40% longer feature development in the billing domain. Proposal: 3-sprint investment to redesign the billing data model, expected payback in 4 months based on velocity data. Risk: payment integration downtime during migration mitigated by feature flags and dual-write."

**Key insight:** Debt is acceptable when it's intentional and tracked. Unintentional/invisible debt is what kills teams. Make it visible with metrics (deployment frequency, lead time, failure rate, MTTR).
