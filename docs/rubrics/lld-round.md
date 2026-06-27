# Low-Level Design (LLD) Rubric

## 4-Tier Scorecard

| Dimension | 1 Strong No-Hire | 2 No-Hire | 3 Lean Hire | 4 Strong Hire |
|-----------|---------------------|-------------|---------------|-----------------|
| **OOP Fundamentals** | Cannot define classes; no encapsulation | Basic classes but no abstraction layers | Proper inheritance/composition; interface segregation | Expert use of SOLID; knows when NOT to use patterns |
| **Design Patterns** | Unaware of patterns | Names patterns but misapplies them | Applies 2-3 patterns correctly with reasoning | Selects patterns based on trade-offs; discusses alternatives |
| **Concurrency** | No awareness of thread safety | Knows locks exist; cannot apply correctly | Correct synchronization; identifies race conditions | Lock-free designs; discusses deadlock prevention strategies |
| **Memory Management** | Memory leaks; dangling pointers | Correct but no ownership model | Clear ownership (RAII/smart ptrs); no leaks | Custom allocators; memory pools; cache-friendly design |
| **Extensibility** | Hard-coded everything | Some parameterization | Plugin/strategy pattern; open for extension | Fully extensible; config-driven; backward-compatible |
| **API Design** | Inconsistent; exposes internals | Functional but coupled | Clean interfaces; low coupling | Self-documenting; versioned; impossible to misuse |

## Key SOLID Signals

| Principle | Good Signal | Bad Signal |
|-----------|------------|------------|
| Single Responsibility | "This class handles only X" | God class doing 5 things |
| Open/Closed | Uses strategy/observer for extension | Modifying existing code for every new feature |
| Liskov Substitution | Subtypes are drop-in replacements | Overridden methods that throw or no-op |
| Interface Segregation | Small, focused interfaces | One interface with 15 methods |
| Dependency Inversion | Depends on abstractions | Concrete class instantiation everywhere |

---

## Problem 1: Design a Rate Limiter

**Level:** SDE 2-3 | **Time:** 25 min

### Requirements
- Support multiple algorithms: fixed window, sliding window, token bucket
- Thread-safe for concurrent requests
- Configurable per-user and global limits
- Return remaining quota in response

### Expected Class Diagram (Text)

```
<<interface>> RateLimiter
  + isAllowed(clientId: str, endpoint: str) -> RateLimitResult
  + getQuota(clientId: str) -> QuotaInfo

TokenBucketLimiter implements RateLimiter
  - buckets: ConcurrentHashMap<str, Bucket>
  - refillRate: float
  - capacity: int

SlidingWindowLimiter implements RateLimiter
  - windows: ConcurrentHashMap<str, Deque<timestamp>>
  - windowSize: Duration
  - maxRequests: int

Bucket
  - tokens: AtomicInteger
  - lastRefill: timestamp
  + tryConsume() -> bool

RateLimitResult
  - allowed: bool
  - remaining: int
  - retryAfter: Duration

RateLimiterFactory
  + create(algorithm: str, config: Config) -> RateLimiter
```

### Expected Interfaces
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional
import time, threading

@dataclass
class RateLimitResult:
    allowed: bool
    remaining: int
    retry_after_ms: Optional[int] = None

class RateLimiter(ABC):
    @abstractmethod
    def is_allowed(self, client_id: str, endpoint: str) -> RateLimitResult:
        pass

class TokenBucketLimiter(RateLimiter):
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.buckets: dict = {}
        self.lock = threading.Lock()
    
    def is_allowed(self, client_id: str, endpoint: str) -> RateLimitResult:
        key = f"{client_id}:{endpoint}"
        with self.lock:
            bucket = self._get_or_create(key)
            bucket.refill()
            if bucket.tokens > 0:
                bucket.tokens -= 1
                return RateLimitResult(True, bucket.tokens)
            return RateLimitResult(False, 0, bucket.ms_until_refill())
```

### What Good Looks Like
- Separates algorithm choice from usage (Strategy pattern)
- Thread safety at the right granularity (per-bucket, not global)
- Discusses distributed rate limiting (Redis, consistent hashing)
- Handles clock skew and race conditions

### What Bad Looks Like
- Single monolithic class with if/else for algorithms
- Global lock for all clients
- No consideration of distributed deployment
- Hard-coded limits with no configuration

---

## Problem 2: Design a Task Scheduler

**Level:** SDE 2-3 | **Time:** 25 min

### Requirements
- Schedule tasks with priority and optional delay
- Worker thread pool executes tasks
- Support recurring tasks (cron-like)
- Graceful shutdown; no task loss
- Observable: task states, queue depth metrics

### Expected Class Diagram (Text)

```
<<interface>> TaskScheduler
  + submit(task: Task) -> TaskHandle
  + schedule(task: Task, delay: Duration) -> TaskHandle
  + scheduleRecurring(task: Task, interval: Duration) -> TaskHandle
  + shutdown(graceful: bool) -> void

<<interface>> Task
  + execute() -> TaskResult
  + getPriority() -> int
  + getTaskId() -> str

TaskHandle
  - taskId: str
  - status: TaskStatus (PENDING|RUNNING|DONE|FAILED)
  + cancel() -> bool
  + await() -> TaskResult

ThreadPoolScheduler implements TaskScheduler
  - queue: PriorityBlockingQueue<ScheduledTask>
  - workers: List<WorkerThread>
  - delayQueue: DelayQueue<ScheduledTask>
  - metrics: SchedulerMetrics

WorkerThread
  - running: AtomicBoolean
  + run() -> void (poll queue, execute, report)

SchedulerMetrics
  - tasksSubmitted: Counter
  - tasksCompleted: Counter
  - tasksFailed: Counter
  - queueDepth: Gauge
  - avgExecutionTime: Histogram
```

### Key Discussion Points

| Topic | Strong Signal | Weak Signal |
|-------|--------------|-------------|
| Priority handling | Priority queue with aging to prevent starvation | Simple FIFO or broken priority |
| Thread safety | Lock-free queue or fine-grained locking | Synchronized everything |
| Shutdown | Drain queue; wait for running tasks; timeout | Kill threads immediately |
| Failure handling | Retry with backoff; dead-letter queue | Silently swallow exceptions |
| Recurring tasks | Re-enqueue after execution with next fire time | Sleep in a loop |
| Observability | Metrics, health checks, task history | No visibility into internal state |

### Follow-up Questions
1. How do you prevent priority inversion/starvation?
2. What happens if a task takes too long? (timeout, circuit breaker)
3. How would you persist tasks across restarts? (WAL, DB-backed queue)
4. How does this differ from a distributed task queue (Celery, SQS)?
