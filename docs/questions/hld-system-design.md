# HLD / System Design Problems

> 25 real system design questions with architecture outlines, trade-offs, and numbers.
> Focus: scalability, reliability, consistency trade-offs, and what separates SDE 3 from SDE 4 answers.

---

### SD1: Design URL Shortener (100M URLs)

**Requirements:**
- Functional: Shorten URL, redirect, analytics (click count), custom aliases, expiration.
- Non-functional: 100M URLs/month, 1000:1 read:write ratio, <100ms redirect latency, 99.9% availability.

**Architecture:**
```
Client → Load Balancer → API Servers → Cache (Redis) → DB (Sharded NoSQL)
                                    → Analytics Pipeline (Kafka → OLAP)
```

**Key Components:**
- ID generation: Base62 encoding of auto-incrementing ID (or pre-generated ID ranges per server).
- Storage: Key-value store (DynamoDB/Cassandra). Key=short_code, Value={long_url, created, expires, user}.
- Cache: Redis with LRU. Cache hot URLs (80/20 rule 20% URLs get 80% traffic).
- Redirection: 301 (permanent, cacheable) vs 302 (temporary, trackable).

**Numbers:** 100M creates/month = ~40 writes/s. 40K reads/s. 500 bytes/URL × 100M = 50GB/month. 5-year: 3TB.

**Trade-offs:** Consistent hashing for DB shards vs range-based partitioning. 301 saves server load but loses analytics.

**SDE 3 vs 4:** SDE 3 covers basic architecture. SDE 4 discusses: ID collision handling at scale, geo-distributed caching, abuse prevention (rate limiting + spam detection), graceful degradation, data retention policies.

---

### SD2: Design News Feed (Facebook Scale)

**Requirements:**
- Functional: Create posts, follow/unfollow, generate feed (ranked), push notifications.
- Non-functional: 500M DAU, feed generation <500ms, eventual consistency OK for feed.

**Architecture:**
```
Write path: Client → LB → Post Service → DB + Fan-out Service → Feed Cache
Read path:  Client → LB → Feed Service → Feed Cache (pre-computed feeds)
```

**Key Components:**
- Fan-out on write: Pre-compute feeds for most users (push model).
- Fan-out on read: For celebrities (1M+ followers), compute at read time (pull model).
- Hybrid: Push for normal users, pull for celebrities.
- Ranking: ML model scores posts. Features: recency, affinity, engagement signals.
- Feed cache: Redis sorted sets per user. Score = ranking score.

**Numbers:** 500M users × avg 500 friends × 10 posts/day = 2.5T feed entries/day. Feed cache: 500M × 1KB = 500TB (only active users cached).

**Trade-offs:** Push vs Pull vs Hybrid. Consistency vs latency. Storage cost vs computation cost.

**SDE 3 vs 4:** SDE 4 adds: celebrity handling, feed diversity (dedup + anti-spam), real-time vs batch ranking, privacy controls at scale, international content delivery.

---

### SD3: Design Chat System (WhatsApp)

**Requirements:**
- Functional: 1:1 chat, group chat (500 max), online/typing indicators, message delivery receipts, media.
- Non-functional: 50M concurrent connections, <100ms message latency, guaranteed delivery.

**Architecture:**
```
Client ←WebSocket→ Connection Server → Message Router → Message Queue → Recipient Connection Server
                                     → Message DB (write-ahead)
                                     → Push Notification (offline users)
```

**Key Components:**
- Connection layer: WebSocket servers. Each maintains user→connection mapping.
- Message ordering: Per-conversation monotonic sequence numbers (Lamport timestamps).
- Offline delivery: Store in queue, deliver when user comes online.
- Group messaging: Fan-out at message router level (not per connection server).
- End-to-end encryption: Signal protocol (key exchange + ratchet).

**Numbers:** 50M connections × ~1KB state = 50GB connection state. 100B messages/day × 100 bytes = 10TB/day. Need ~50K WebSocket servers (1K connections each after optimization).

**SDE 3 vs 4:** SDE 4 discusses: message ordering guarantees across devices, E2E encryption key management, cross-region message delivery, media upload/CDN integration, presence system at scale.

---

### SD4: Design Video Streaming (Netflix)

**Requirements:**
- Functional: Upload, transcode, stream (adaptive bitrate), recommendations, resume playback.
- Non-functional: 200M subscribers, 10PB content, <2s startup time, 99.99% availability.

**Architecture:**
```
Upload: Creator → Upload Service → Object Store → Transcoding Pipeline → CDN
Stream: Client → CDN (edge) → Origin (if miss) → Object Store
Control: Client → API → Recommendation Engine + User Profile DB
```

**Key Components:**
- Transcoding: Multiple bitrates/codecs (H.264, VP9, AV1). Parallel chunk processing.
- Adaptive Bitrate: Client measures bandwidth, requests appropriate quality (DASH/HLS manifests).
- CDN: Open Connect Appliances (Netflix's own). Predictive pre-positioning content.
- Recommendation: Collaborative filtering + content-based + session context.

**Numbers:** 10PB × multiple encodings = ~100PB storage. Peak: 200M * 10% concurrent = 20M streams. ~40Tbps bandwidth.

**SDE 3 vs 4:** SDE 4 adds: codec selection per device, bandwidth prediction models, prefetching logic, DRM, live streaming architecture, cold start content positioning.

---

### SD5: Design Ride Sharing (Uber)

**Requirements:**
- Functional: Request ride, match driver, real-time tracking, pricing, payments, rating.
- Non-functional: <30s match time, real-time location updates, high availability.

**Architecture:**
```
Rider App → API Gateway → Trip Service → Matching Engine → Driver App
                       → Location Service (Redis Geo) → ETA Engine
                       → Pricing Service (surge) → Payment Service
```

**Key Components:**
- Location service: Redis GeoSpatial or S2 geometry cells. Index drivers by geo-cell.
- Matching: Filter nearby drivers → rank by ETA/rating → offer to best.
- Surge pricing: Supply-demand ratio per geo-zone. Dynamic multiplier.
- ETA: Graph-based routing (Dijkstra/A* with real-time traffic overlay).

**Numbers:** 5M rides/day = 60 rides/s. 2M active drivers × location update/4s = 500K updates/s.

**SDE 3 vs 4:** SDE 4 adds: geo-cell granularity optimization, driver positioning incentives, multi-destination trip planning, fraud detection, marketplace economics modeling.

---

### SD6: Design Notification System (10M Users)

**Requirements:**
- Functional: Push notifications (mobile), email, SMS, in-app. Priority levels, templates, preferences.
- Non-functional: 10M users, 1B notifications/day, at-least-once delivery, <5s latency for critical.

**Architecture:**
```
Event Source → Notification Service → Priority Queue → Delivery Workers → Provider APIs
                                   → Template Engine     (APNS, FCM, SES, Twilio)
                                   → Preference Check
                                   → Dedup/Rate Limit
```

**Key Components:**
- Priority queues: Critical (immediate), High (< 1min), Low (batched).
- Rate limiting: Per-user per-channel limits (no spam).
- Dedup: Idempotency keys to prevent duplicate sends.
- Retry: Exponential backoff for provider failures.
- Analytics: Track delivered, opened, clicked.

**Numbers:** 1B/day = ~12K/s. Peak: 50K/s. Provider rate limits: APNS ~100K/s, FCM ~500K/s.

**SDE 3 vs 4:** SDE 4 adds: intelligent batching (digest mode), cross-channel orchestration, A/B testing of content, quiet hours per timezone, unsubscribe compliance.

---

### SD7: Design Distributed Rate Limiter

**Requirements:**
- Functional: Per-client rate limiting across multiple API servers. Configurable rules.
- Non-functional: <1ms overhead per request, eventually consistent counts, 99.99% availability.

**Architecture:**
```
API Server → Local Counter (approximate) → Sync with Redis/Central Store
             ↓ (async)
         Rate Limit Config Service
```

**Key Components:**
- Algorithm: Token bucket (bursty OK) or sliding window log (precise).
- Local + remote hybrid: Sync every N requests or T seconds.
- Redis implementation: `MULTI/EXEC` or Lua script for atomicity.
- Failover: Default to allow (fail-open) or deny (fail-closed) when Redis is down.

**Trade-offs:** Precision vs performance. Fail-open (availability) vs fail-closed (protection).

---

### SD8: Design CDN

**Requirements:**
- Functional: Cache static/dynamic content at edge, purge, origin pull, HTTPS termination.
- Non-functional: Global coverage, <50ms latency P50, 99.99% availability.

**Architecture:**
```
Client → DNS (GeoDNS) → Edge PoP → Regional Cache → Origin
                              ↓
                     TLS Termination + Compression
```

**Key Components:**
- DNS routing: Anycast or GeoDNS to nearest PoP.
- Cache hierarchy: Edge → Regional → Origin. Reduces origin load.
- Invalidation: Purge API + TTL-based. Versioned URLs for immutable content.
- Consistent hashing: Distribute content across edge servers in a PoP.

---

### SD9: Design Search Autocomplete

**Requirements:**
- Functional: Type-ahead suggestions (top 10), personalized, freshness.
- Non-functional: <100ms P99, handles 100K QPS, trilions of historical queries.

**Architecture:**
```
Client → LB → Autocomplete Service → Trie Cache (Redis/In-memory)
                                    ← Data Pipeline (offline Trie rebuild)
Data Collection → Kafka → Aggregator → Trie Builder → Deploy to edge
```

**Key Components:**
- Trie: Each node stores top-K completions (pre-computed).
- Ranking: Frequency × recency × personalization score.
- Update: Batch rebuild every 15 min. Not real-time (too expensive).
- Sharding: By prefix (a-m on shard 1, n-z on shard 2) or consistent hash.

---

### SD10: Design Payment System

**Requirements:**
- Functional: Process payments, refunds, ledger, multi-currency, reconciliation.
- Non-functional: ACID for transactions, idempotent APIs, PCI DSS compliance.

**Architecture:**
```
Client → API Gateway → Payment Service → Payment State Machine → PSP (Stripe/Adyen)
                    → Ledger Service (double-entry)
                    → Fraud Detection (ML)
                    → Reconciliation (batch)
```

**Key Components:**
- Idempotency: Client-generated idempotency key. Dedup on server.
- State machine: Created → Authorized → Captured → Settled (or Failed/Refunded).
- Double-entry ledger: Every transaction has debit + credit entries.
- Reconciliation: Compare internal ledger with PSP settlements daily.

**SDE 3 vs 4:** SDE 4 adds: multi-PSP failover, currency conversion timing, regulatory compliance (PSD2/SCA), partial captures, chargeback handling.

---

### SD11: Design Distributed Cache (Redis Clone)

**Requirements:**
- Functional: GET/SET/DEL, TTL, LRU eviction, pub/sub, data structures.
- Non-functional: <1ms P99, millions of ops/sec, horizontal scaling.

**Key Components:**
- Single-threaded event loop (like Redis): Avoids lock overhead.
- Data structures: Hash table for KV, skip list for sorted sets.
- Persistence: RDB snapshots + AOF append-only file.
- Clustering: Hash slots (16384 slots, assigned to nodes). Client-side routing.
- Replication: Async replica for reads. Failover with Sentinel/Raft.

---

### SD12: Design Message Queue (Kafka)

**Requirements:**
- Functional: Publish, subscribe, consumer groups, replay, ordering per partition.
- Non-functional: 1M msgs/s, <10ms P99, durable (no message loss), exactly-once semantics.

**Key Components:**
- Partitioned log: Topic → N partitions. Each partition is an ordered, append-only log.
- Consumer groups: Each partition consumed by exactly one consumer in a group.
- Replication: ISR (in-sync replicas). Ack after leader + N-1 replicas.
- Compaction: Log compaction for changelog topics (keep latest per key).
- Zero-copy: sendfile syscall for high-throughput delivery.

---

### SD13: Design Social Graph

**Requirements:**
- Functional: Follow/unfollow, friends of friends, mutual connections, recommendations.
- Non-functional: Billions of edges, <100ms for 2-hop queries.

**Key Components:**
- Storage: Adjacency list in distributed graph DB (TAO/Neo4j) or wide-column store.
- Caching: Materialize friend lists in cache. Invalidate on follow/unfollow.
- BFS/graph queries: Pre-computed for common patterns (mutual friends).
- Sharding: By user_id. Cross-shard edges require scatter-gather.

---

### SD14: Design E-Commerce Cart

**Requirements:**
- Functional: Add/remove items, apply coupons, inventory reservation, checkout.
- Non-functional: Handle flash sales (100K RPS), inventory consistency.

**Key Components:**
- Cart storage: Redis (session-based) + persist to DB on checkout.
- Inventory: Soft reservation on add-to-cart (TTL-based). Hard reservation on checkout.
- Pricing: Calculate server-side (never trust client). Apply promotions engine.
- Checkout: Saga pattern (reserve inventory → charge payment → confirm order).

---

### SD15: Design Metrics/Monitoring System

**Requirements:**
- Functional: Ingest metrics (counters, gauges, histograms), query, alert, dashboard.
- Non-functional: 10M metrics/s ingestion, 1-min query latency for 30-day range.

**Architecture:**
```
Agents → Collector (Kafka buffer) → Storage (time-series DB: blocks + compaction)
Query Engine ← Dashboards/Alerts
```

**Key Components:**
- Time-series DB: Append-only, compressed blocks (gorilla encoding for timestamps, XOR for values).
- Downsampling: 1s resolution → 1min → 1hr for older data.
- Alerting: Streaming evaluation of rules against latest data.
- Labels/tags: Inverted index for efficient multi-dimensional queries.

---

### SD16: Design OTA Firmware Update (IoT)

**Requirements:**
- Functional: Upload firmware, staged rollout, delta updates, rollback, device targeting.
- Non-functional: 10M devices, unreliable connections, minimal bandwidth.

**Key Components:**
- Delta updates: Binary diff (bsdiff) to minimize download size.
- Staged rollout: 1% → 5% → 25% → 100%. Automatic pause on error spike.
- A/B partition: Device has two firmware slots. Boot into new; rollback to old if unhealthy.
- Resume: Chunked download with offset tracking for interrupted connections.

---

### SD17: Design Flight Booking System

**Requirements:**
- Functional: Search flights, seat selection, booking, payment, cancellation.
- Non-functional: Strong consistency for seat inventory, handle 10K bookings/min.

**Key Components:**
- Search: Pre-computed fare classes. Cached availability. Eventual consistency OK.
- Booking: Pessimistic locking on seat inventory during checkout (short TTL lock).
- Saga: Search → Reserve (5min lock) → Pay → Confirm. Compensate on failure.
- Pricing: Dynamic pricing engine. Fare class buckets.

---

### SD18: Design Real-Time Gaming Leaderboard

**Requirements:**
- Functional: Submit score, get rank, get top-K, get nearby players.
- Non-functional: 1M players, real-time updates, <50ms query.

**Key Components:**
- Redis Sorted Set: O(log N) add, O(log N) rank query. `ZADD`, `ZRANK`, `ZRANGE`.
- Sharding for billions: Partition by score range. Query top-K from top shard.
- Periodic snapshots: Freeze leaderboard for seasons/events.
- Approximate rank for global: Count players in higher score buckets.

---

### SD19: Design Collaborative Document Editor

**Requirements:**
- Functional: Real-time co-editing, cursor presence, version history, offline support.
- Non-functional: <100ms sync latency, conflict-free concurrent edits.

**Key Components:**
- CRDT (Conflict-free Replicated Data Types): Or OT (Operational Transformation).
- WebSocket: Bidirectional sync for real-time updates.
- Operation log: Append-only log of all edits. Reconstruct any version.
- Presence: Broadcast cursor positions via pub/sub.

**Trade-offs:** OT (Google Docs, centralized) vs CRDT (Yjs, decentralized). OT simpler server but requires central authority.

---

### SD20: Design API Gateway

**Requirements:**
- Functional: Routing, auth, rate limiting, request transformation, load balancing.
- Non-functional: <5ms added latency, 100K RPS, zero-downtime deploys.

**Key Components:**
- Plugin architecture: Auth → Rate limit → Transform → Route → Backend.
- Config: Dynamic routing rules (reload without restart).
- Circuit breaking: Per-backend health tracking.
- Observability: Request logging, distributed tracing headers.

---

### SD21: Design Log Aggregation (ELK)

**Requirements:**
- Functional: Collect logs from 10K servers, search, filter, alerting.
- Non-functional: 100GB logs/day, <30s ingest-to-search latency, 30-day retention.

**Architecture:**
```
Apps → Filebeat/Fluentd → Kafka (buffer) → Logstash (parse/transform) → Elasticsearch
                                                                      → Cold Storage (S3)
Dashboard/Alerting ← Kibana/Grafana
```

**Key Components:**
- Index per day for easy rotation. ILM (Index Lifecycle Management) for hot→warm→cold.
- Schema: structured JSON logs. Parse at ingest (not query time).
- Alerting: Elastalert or Watcher for pattern detection.

---

### SD22: Design Distributed Task Scheduler

**Requirements:**
- Functional: Schedule tasks (cron + one-shot), ensure exactly-once execution, retries.
- Non-functional: 100K tasks/minute, no missed executions, fault-tolerant.

**Key Components:**
- Task store: DB with status (pending/running/done/failed) + next_run_at.
- Workers: Poll for ready tasks. Claim with optimistic locking (UPDATE WHERE status=pending).
- Leader: One scheduler process computes next-run times. Workers just execute.
- Dead worker detection: Heartbeat-based. Reclaim timed-out tasks.

---

### SD23: Design Content Moderation Pipeline

**Requirements:**
- Functional: Review text/image/video content. Auto-flag, human review, appeal.
- Non-functional: Process 100M posts/day, <2s for auto-moderation, <24h human review.

**Architecture:**
```
Content Upload → ML Classifier (text/image/video) → Confidence Score
  High confidence bad → Auto-remove + notify
  Medium confidence → Human review queue (priority ranked)
  Low confidence → Allow (sample for QA)
```

**Key Components:**
- ML models: Toxicity, nudity, violence, spam. Ensemble for higher accuracy.
- Human review: Prioritized queue. Reviewer consensus (2-of-3 agree).
- Appeal flow: Re-review by senior moderator. Track false positive rate.
- Feedback loop: Human decisions retrain ML models.

---

### SD24: Design Feature Flag System

**Requirements:**
- Functional: Toggle features per user/segment/percentage. A/B testing. Kill switch.
- Non-functional: <5ms evaluation, 100% availability (fail-open), real-time toggle.

**Architecture:**
```
Admin UI → Flag Config Store (DB) → Push to edge SDKs (WebSocket/SSE)
Application → SDK (local cache) → Evaluate flag rules
                               → Report events to analytics
```

**Key Components:**
- Evaluation engine: Rules engine (user.country == "US" AND user.tier == "premium").
- Percentage rollout: Hash(user_id + flag_name) % 100 < percentage.
- Local cache: SDK caches all flags. Streaming updates from server.
- Fallback: If evaluation service is down, use cached values or defaults.

---

### SD25: Design Multi-Region Database

**Requirements:**
- Functional: Read/write from any region, cross-region consistency, failover.
- Non-functional: <100ms local reads, <500ms cross-region writes, RPO=0 for primary.

**Architecture:**
```
Region A (Primary): Write → Local DB → Sync Replication to Region B
Region B (Secondary): Read → Local Replica (async lag < 1s)
                      Write → Forward to Region A (or local if active-active)
```

**Key Components:**
- Active-passive: Single write region. Read replicas in other regions.
- Active-active: Multi-master (CockroachDB/Spanner style). Conflict resolution needed.
- Consensus: Paxos/Raft across regions for strong consistency (high latency writes).
- Conflict resolution: Last-writer-wins, vector clocks, or application-level merge.

**Trade-offs:**

| Approach | Write Latency | Consistency | Complexity |
|----------|--------------|-------------|------------|
| Active-passive | High (cross-region) | Strong | Low |
| Active-active (LWW) | Low | Eventual | Medium |
| Active-active (Consensus) | High | Strong | High |

**SDE 3 vs 4:** SDE 4 discusses: split-brain handling, observability of replication lag, failover automation, data sovereignty (GDPR), and when to sacrifice consistency for availability.
