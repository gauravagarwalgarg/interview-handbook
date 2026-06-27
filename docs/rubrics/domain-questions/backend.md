# Backend & Cloud Domain Questions

## Distributed Systems

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 1 | How would you ensure exactly-once message processing in a distributed queue? | SDE 3 | Idempotency keys + dedup store (Redis/DB); acknowledge after processing; consumer offset management. At-least-once delivery + idempotent handlers is the practical approach. | What's the difference between exactly-once delivery vs processing? How do you handle dedup store failures? |
| 2 | Explain the CAP theorem with a real-world example. When would you choose AP over CP? | SDE 2 | CAP: can't have Consistency + Availability + Partition tolerance simultaneously. AP example: social media feed (eventual consistency OK). CP example: banking transactions. | What is PACELC? How does DynamoDB handle this with tunable consistency? |
| 3 | How do you design a system to handle 100K writes/sec with strong consistency? | SDE 3-4 | Partitioned writes (consistent hashing); WAL for durability; Raft/Paxos for consensus within partition; batch writes. Discuss: leader-based replication per shard. | What's the write amplification cost? How do you handle hot partitions? |
| 4 | What happens when a network partition occurs in a Kafka cluster? | SDE 3 | ISR (In-Sync Replicas) shrinks; leader continues if min.insync.replicas met; otherwise producer gets NotEnoughReplicas. Consumers may see lag. | How does unclean.leader.election affect data loss? How do you monitor for this? |

## Databases

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 5 | When would you choose a document store over a relational DB? | SDE 2 | Document store: schema flexibility, hierarchical data, read-heavy denormalized access patterns. Relational: complex joins, ACID transactions, normalized data with relationships. | How do you handle transactions in MongoDB? What about reporting queries on document stores? |
| 6 | Explain database sharding strategies and their trade-offs. | SDE 3 | Range-based (hotspots but range queries), hash-based (even distribution but no range queries), directory-based (flexible but extra lookup). Cross-shard joins are expensive. | How do you handle resharding? What about cross-shard transactions? |
| 7 | How do you handle schema migrations on a database with 99.99% uptime SLA? | SDE 3 | Expand-contract pattern: add new column → backfill → migrate readers → drop old column. Never do breaking changes in one step. Use online DDL tools (gh-ost, pt-osc). | How do you handle failed migrations? What about NoSQL schema evolution? |

## Caching

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 8 | Explain cache invalidation strategies. When does each fail? | SDE 2-3 | TTL-based (stale data window), event-based (complexity), write-through (latency on write), write-behind (data loss risk). Cache-aside is most common: read miss → load → cache. | How do you handle thundering herd? What about cache stampede on hot keys? |
| 9 | Design a multi-layer caching strategy for a high-traffic e-commerce site. | SDE 3 | CDN (static assets) → Application cache (Redis: sessions, product catalog) → Database query cache → Connection pooling. Invalidation: event-driven from DB change stream. | How do you handle personalization with CDN caching? Cache warming strategies? |

## API Design

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 10 | How do you design a REST API that needs to support backward compatibility? | SDE 2 | URL versioning (/v1/, /v2/) or header-based. Additive changes only within version. Deprecation policy with sunset headers. Never remove fields; add new ones. | REST vs GraphQL for this problem? How do you handle breaking changes in gRPC (protobuf)? |
| 11 | Explain rate limiting algorithms and when to use each. | SDE 2-3 | Token bucket (burst-friendly), leaky bucket (smooth output), fixed window (boundary issues), sliding window log (precise but memory-heavy), sliding window counter (good balance). | How do you implement distributed rate limiting? How do you handle rate limit fairness across tenants? |

## Message Queues & Event-Driven Architecture

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 12 | Compare Kafka vs RabbitMQ vs SQS. When would you pick each? | SDE 2-3 | Kafka: high-throughput streaming, replay, ordering per partition. RabbitMQ: complex routing, lower latency, traditional pub/sub. SQS: managed, simpler, auto-scaling consumers. | How does Kafka handle consumer rebalancing? What are the gotchas with SQS FIFO queues? |
| 13 | How do you implement the Saga pattern for distributed transactions? | SDE 3 | Orchestration (central coordinator) vs Choreography (event-driven). Compensating transactions for rollback. State machine for saga progress. Timeout + dead letter for stuck sagas. | How do you handle partial failures in choreography? What about observability of saga state? |

## Microservices

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 14 | How do you handle service discovery and load balancing in microservices? | SDE 2-3 | Client-side (Eureka + ribbon) vs server-side (ALB/Nginx) vs service mesh (Envoy/Istio). DNS-based (simple but TTL issues). Health checks for instance removal. | How does a service mesh handle circuit breaking? What's the overhead of a sidecar proxy? |
| 15 | Describe the circuit breaker pattern. Implement the state machine. | SDE 2 | States: Closed (normal) → Open (failing, reject fast) → Half-Open (test recovery). Track failure rate over window; trip at threshold. Half-open allows N probe requests. | How do you set thresholds? How does this interact with retries and timeouts? What about bulkhead isolation? |
