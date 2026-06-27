# High-Level Design (HLD) Rubric

## Cloud Systems 4-Tier Scorecard

| Dimension | 1 Strong No-Hire | 2 No-Hire | 3 Lean Hire | 4 Strong Hire |
|-----------|---------------------|-------------|---------------|-----------------|
| **Scalability** | Single-server thinking | Mentions scaling but no concrete strategy | Horizontal scaling with load balancers; stateless services | Auto-scaling policies; capacity planning with numbers |
| **Database Design** | Single table; no indexing thought | Normalized schema; no sharding awareness | Sharding strategy; read replicas; query patterns | Multi-model (SQL+NoSQL); partition key design; migration plan |
| **Caching** | No caching layer | "Add Redis" without strategy | Cache-aside/write-through with TTL reasoning | Multi-layer cache; invalidation strategy; thundering herd mitigation |
| **CAP Theorem** | Unaware | Can state CAP but not apply | Chooses CP/AP with reasoning for use case | Designs for tunable consistency; discusses PACELC |
| **API Design** | No API structure | REST endpoints but no versioning | RESTful with pagination, rate limiting, versioning | API gateway; GraphQL federation; backward-compatible evolution |
| **Message Queues** | Synchronous everything | Knows queues exist; no partitioning thought | Appropriate async boundaries; DLQ; retry | Ordering guarantees; exactly-once semantics; backpressure |
| **Observability** | No monitoring discussion | "Add logs" | Structured logging, metrics, alerting | Distributed tracing; SLO-based alerts; runbooks |

## Embedded Systems 4-Tier Scorecard

| Dimension | 1 Strong No-Hire | 2 No-Hire | 3 Lean Hire | 4 Strong Hire |
|-----------|---------------------|-------------|---------------|-----------------|
| **RTOS Scheduling** | No awareness of real-time constraints | Knows RTOS exists; can't discuss scheduling | Rate-monotonic; priority-based preemption; deadline analysis | Mixed criticality; WCET analysis; priority ceiling protocol |
| **Interrupt Design** | No ISR awareness | Basic ISR but lengthy processing in handler | Top-half/bottom-half split; proper prioritization | Nested interrupts; latency budgeting; DMA offloading |
| **Memory Architecture** | No awareness of constraints | Knows RAM is limited | Memory pools; static allocation; no heap in ISR | MPU regions; cache coherency; scatter-gather DMA |
| **Low-Power Design** | Always-on thinking | Knows sleep modes exist | Sleep/wake state machine; peripheral clock gating | Power domains; energy harvesting budget; duty cycle optimization |
| **Communication** | Single protocol awareness | Knows SPI/I2C/UART basics | Protocol selection with trade-offs; DMA transfers | Multi-bus architecture; arbitration; error recovery protocols |
| **Safety/Reliability** | No fault consideration | Basic error checking | Watchdog; CRC; redundancy for critical paths | FMEA-driven design; fail-safe states; diagnostic coverage |

---

## Problem 1 (Cloud): Design a Notification System

**Level:** SDE 3-4 | **Scale:** 10M users, real-time delivery

### Requirements
- Multi-channel: push, email, SMS, in-app
- Real-time (<1s for push); eventual for email/SMS
- User preferences (opt-in/out per channel per category)
- Rate limiting per user (no spam)
- Delivery guarantees with retry
- Analytics: open rates, click-through

### Expected Architecture

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐
│ API Gateway │───▶│ Notification │───▶│  Priority Queue  │
│ (Rate Limit)│    │   Service    │    │  (Kafka/SQS)     │
└─────────────┘    └──────────────┘    └────────┬────────┘
                          │                      │
                   ┌──────▼──────┐        ┌──────▼──────┐
                   │ Preference  │        │  Dispatcher  │
                   │   Service   │        │  (Workers)   │
                   │ (Redis/DB)  │        └──┬───┬───┬──┘
                   └─────────────┘           │   │   │
                                      ┌──────┘   │   └──────┐
                                      ▼          ▼          ▼
                                   [Push]    [Email]     [SMS]
                                   (FCM/     (SES/       (Twilio/
                                    APNs)    SendGrid)    SNS)
```

### Key Design Decisions

| Decision | Options | Recommended | Reasoning |
|----------|---------|-------------|-----------|
| Queue | Kafka vs SQS | Kafka | Ordering per user; high throughput; replay |
| Preference store | SQL vs Redis | Redis + SQL | Redis for hot path; SQL for durability |
| Push delivery | Direct vs fan-out | Topic per device type | Reduces connection overhead |
| Dedup | Application vs broker | Application (idempotency key) | Control over window |
| Rate limiting | Per-service vs centralized | Centralized (Redis) | Consistent across channels |

### Capacity Estimation
- 10M users × 5 notifications/day = 50M/day ≈ 580/sec avg, 5K/sec peak
- Push payload: ~1KB → 5GB/day egress
- Kafka: 3 partitions per channel; replication factor 3

### Follow-up Questions
1. How do you handle a provider outage (FCM down)?
2. How do you ensure exactly-once delivery for payments notifications?
3. How would you A/B test notification content at scale?
4. Design the analytics pipeline for open/click tracking.

### Scoring Guide
| Score | Signal |
|-------|--------|
| 1 | Synchronous delivery; no queue; single-channel |
| 2 | Queue-based but no scaling numbers; single provider |
| 3 | Multi-channel with preferences; basic capacity math |
| 4 | Full architecture with failure modes, capacity, analytics, cost |

---

## Problem 2 (Embedded): Design an OTA Firmware Update System

**Level:** SDE 3-4 | **Scale:** 50K IoT devices, unreliable connectivity

### Requirements
- Secure: signed firmware, encrypted transport, rollback on failure
- Resumable: handle network drops mid-transfer
- Staged rollout: canary → gradual → full fleet
- Minimal downtime: A/B partition scheme
- Bandwidth-efficient: delta updates where possible
- Monitoring: fleet-wide update status dashboard

### Expected Architecture

```
┌────────────────────────────────────────────────────┐
│                  Cloud Backend                       │
│  ┌──────────┐  ┌───────────┐  ┌────────────────┐  │
│  │ Release  │  │  Campaign  │  │  Device Shadow  │  │
│  │ Manager  │  │  Manager   │  │  (State Store)  │  │
│  └────┬─────┘  └─────┬─────┘  └───────┬────────┘  │
│       │               │                │            │
│  ┌────▼───────────────▼────────────────▼────┐      │
│  │           CDN / Artifact Store            │      │
│  └────────────────────┬─────────────────────┘      │
└───────────────────────┼─────────────────────────────┘
                        │ HTTPS/MQTT (TLS 1.3)
┌───────────────────────▼─────────────────────────────┐
│                  IoT Device                           │
│  ┌──────────┐  ┌───────────┐  ┌────────────────┐   │
│  │  Update  │  │ Partition  │  │   Bootloader   │   │
│  │  Agent   │  │  Manager   │  │  (Verified Boot)│   │
│  └────┬─────┘  └─────┬─────┘  └───────┬────────┘   │
│       │     Download  │  Swap           │ Verify     │
│       ▼───────────────▼────────────────▼            │
│  [Slot A (active)] ←→ [Slot B (staging)]            │
└──────────────────────────────────────────────────────┘
```

### A/B Partition Update Flow

| Step | Action | Failure Recovery |
|------|--------|-----------------|
| 1 | Device polls/receives update notification | Retry with exponential backoff |
| 2 | Download firmware to inactive slot (B) | Resume from last byte offset |
| 3 | Verify signature (RSA-2048 / Ed25519) | Reject; report tampered artifact |
| 4 | Mark slot B as "pending verification" | Reboot to slot A if B fails |
| 5 | Reboot into slot B | Watchdog triggers rollback to A |
| 6 | Health check passes → mark B as "good" | 3 retries before rollback |
| 7 | Report success to cloud | Retry report; device shadow sync |

### Staged Rollout Strategy

| Phase | Fleet % | Duration | Gate Criteria |
|-------|---------|----------|---------------|
| Canary | 1% (500 devices) | 24h | 0 rollbacks, <2% error rate |
| Early | 10% (5K devices) | 48h | <0.5% rollback rate |
| Majority | 50% | 72h | Monitoring stable |
| Full | 100% | | Auto-promote if gates pass |

### Key Discussion Points

| Topic | Strong Signal | Weak Signal |
|-------|--------------|-------------|
| Security | Chain of trust; secure boot; key rotation | "Just use HTTPS" |
| Resilience | A/B slots; watchdog; health checks | Single partition; brick risk |
| Bandwidth | Delta updates (bsdiff); compression; CDN | Full image every time |
| Fleet mgmt | Staged rollout; automatic rollback gates | Push to all simultaneously |
| Power safety | Atomic commit; power-loss-safe writes | Corruption on power cut |

### Follow-up Questions
1. How do you handle a device that's been offline for 6 months (multiple versions behind)?
2. How do you revoke a compromised signing key across the fleet?
3. What's your watchdog timeout strategy during first boot of new firmware?
4. How do you handle mixed hardware revisions needing different firmware?
