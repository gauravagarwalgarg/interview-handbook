# Concurrency  Deep Dive

**Focus: Threading Models, Atomics, Lock-Free Design, Memory Ordering, Real-Time Constraints**

---

## 🔬 Q1: Producer-Consumer for Telemetry Pipeline

**Question:** You're building a telemetry pipeline for a satellite. A high-priority ISR produces sensor samples at 1 kHz. A lower-priority thread consumes and packetizes them for downlink. Design the inter-thread communication.

**Constraints:**
- ISR must not block (no mutex, no allocation)
- Consumer may occasionally fall behind (handle overflow gracefully)
- ARM Cortex-A53, Linux with PREEMPT_RT patch

**Expected answer: Lock-free SPSC ring buffer**

```cpp
template <typename T, std::size_t Capacity>
class SpscRingBuffer {
    static_assert((Capacity & (Capacity - 1)) == 0, "Capacity must be power of 2");

public:
    bool try_push(const T& item) noexcept {
        const auto head = head_.load(std::memory_order_relaxed);
        const auto next = (head + 1) & (Capacity - 1);
        if (next == tail_.load(std::memory_order_acquire)) {
            return false;  // Full  overflow
        }
        buffer_[head] = item;
        head_.store(next, std::memory_order_release);
        return true;
    }

    bool try_pop(T& item) noexcept {
        const auto tail = tail_.load(std::memory_order_relaxed);
        if (tail == head_.load(std::memory_order_acquire)) {
            return false;  // Empty
        }
        item = buffer_[tail];
        tail_.store((tail + 1) & (Capacity - 1), std::memory_order_release);
        return true;
    }

private:
    alignas(64) std::array<T, Capacity> buffer_{};
    alignas(64) std::atomic<std::size_t> head_{0};
    alignas(64) std::atomic<std::size_t> tail_{0};
};
```

**Follow-up probes:**
- Why `alignas(64)`? (Prevent false sharing  separate cache lines for head/tail)
- Why power-of-2 capacity? (Bitwise AND replaces expensive modulo)
- What memory ordering would you use on x86 vs. ARM? (x86 has strong ordering  `relaxed` is effectively `seq_cst` for stores. ARM is weakly ordered  orderings matter.)
- How would you signal the consumer? (`eventfd`, `futex`, or `sem_post` from ISR context)

---

## 🔬 Q2: Deadlock Analysis

**Question:** Given this code running in an RTOS with priority inheritance:

```cpp
std::mutex mtx_a, mtx_b;

void task_high_priority() {    // Priority 10
    std::lock_guard<std::mutex> lock_a(mtx_a);
    // ... work ...
    std::lock_guard<std::mutex> lock_b(mtx_b);
}

void task_low_priority() {     // Priority 5
    std::lock_guard<std::mutex> lock_b(mtx_b);
    // ... work ...
    std::lock_guard<std::mutex> lock_a(mtx_a);
}
```

1. Can this deadlock? Under what scheduling scenario?
2. How do you fix it?
3. Does priority inheritance help here?

**Expected answers:**

1. **Yes.** Classic ABBA deadlock: High acquires A, Low acquires B, High blocks on B, Low blocks on A.

2. **Fixes:**
   - Consistent lock ordering (always A before B)
   - `std::scoped_lock(mtx_a, mtx_b)`  uses deadlock avoidance algorithm (C++17)
   - Replace with a single mutex if critical sections can be merged
   - Lock-free design if possible

3. **Priority inheritance does NOT prevent deadlock.** It prevents priority inversion (a medium-priority task preempting the low-priority lock holder). The deadlock still occurs because both tasks are mutually waiting.

---

## 🔬 Q3: std::atomic and Memory Ordering on ARM

**Question:** Explain what is wrong with this flag-based synchronization on an ARM Cortex-A53:

```cpp
int data = 0;
std::atomic<bool> ready{false};

// Thread 1 (Writer)
void producer() {
    data = 42;
    ready.store(true, std::memory_order_relaxed);  // BUG
}

// Thread 2 (Reader)
void consumer() {
    while (!ready.load(std::memory_order_relaxed)) {}  // BUG
    assert(data == 42);  // May fire!
}
```

**Expected answer:**
- `memory_order_relaxed` provides no ordering guarantees between `data` and `ready`.
- On ARM (weakly-ordered architecture), the store to `data` may become visible to Thread 2 *after* the store to `ready`.
- **Fix:** Use `memory_order_release` on the store and `memory_order_acquire` on the load. This creates a happens-before relationship.

```cpp
ready.store(true, std::memory_order_release);   // Writer
while (!ready.load(std::memory_order_acquire)) {}  // Reader
// data = 42 is now guaranteed visible
```

**Follow-up:** What hardware instructions does ARM emit for acquire/release? (`LDAR`/`STLR` on ARMv8, or `DMB` barriers on ARMv7)

---

## 🔬 Q4: Thread Pool for Embedded Linux

**Question:** Design a thread pool for a multi-sensor fusion application on embedded Linux. Requirements:
- Fixed number of worker threads (pinned to specific CPU cores)
- Priority-based task queue (high-priority sensor fusion runs before logging)
- Graceful shutdown without data loss
- No dynamic allocation after initialization

**Key evaluation points:**
- `std::priority_queue` with custom comparator for task ordering
- `std::condition_variable` for worker wake-up
- `pthread_setaffinity_np` for core pinning
- Poison pill or atomic flag for shutdown
- Pre-allocated task slots (object pool)

```cpp
class ThreadPool {
public:
    explicit ThreadPool(std::size_t num_threads) : stop_(false) {
        workers_.reserve(num_threads);
        for (std::size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this, i] { worker_loop(i); });
        }
    }

    void submit(Task task, Priority prio) {
        {
            std::lock_guard lock(mtx_);
            tasks_.push({std::move(task), prio});
        }
        cv_.notify_one();
    }

    ~ThreadPool() {
        {
            std::lock_guard lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
        for (auto& w : workers_) w.join();
    }

private:
    void worker_loop(std::size_t core_id) {
        pin_to_core(core_id);
        while (true) {
            PrioritizedTask task;
            {
                std::unique_lock lock(mtx_);
                cv_.wait(lock, [this] { return stop_ || !tasks_.empty(); });
                if (stop_ && tasks_.empty()) return;
                task = std::move(const_cast<PrioritizedTask&>(tasks_.top()));
                tasks_.pop();
            }
            task.func();
        }
    }

    std::vector<std::thread> workers_;
    std::priority_queue<PrioritizedTask> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_;
};
```

---

## 🔬 Q5: Real-Time Constraints and `std::mutex`

**Question:** Why can't you use `std::mutex` in a hard real-time context (e.g., ARINC 653 partition)? What alternatives exist?

**Expected answer:**

`std::mutex` problems in real-time:
1. **Unbounded blocking time**  no guarantee on when the mutex becomes available
2. **Priority inversion**  `std::mutex` does not mandate priority inheritance (implementation-defined)
3. **No timeout**  `std::mutex::lock()` blocks indefinitely (use `std::timed_mutex` for bounded wait)
4. **Kernel involvement**  futex-based implementations may enter kernel, causing non-deterministic latency

**Alternatives:**
| Technique | Use Case |
|-----------|----------|
| Lock-free atomics | Simple shared flags/counters |
| SPSC queues | ISR-to-thread communication |
| Priority-ceiling protocol | RTOS mutex with ceiling priority |
| Interrupt disable | Single-core MCU critical sections |
| `pthread_mutexattr_setprotocol(PTHREAD_PRIO_INHERIT)` | Linux PREEMPT_RT with priority inheritance |
| Sequence locks (seqlock) | Read-heavy, single writer |

---

## ⚡ Quick Concurrency Checks

| Question | Key Answer |
|----------|-----------|
| What is `std::jthread` (C++20)? | Auto-joining thread with cooperative cancellation via `stop_token` |
| Difference between `std::async` and `std::thread`? | `async` returns `std::future`, may run deferred; `thread` always spawns |
| What is a spurious wakeup? | `condition_variable::wait()` can return without `notify`  always use predicate |
| What does `std::call_once` guarantee? | Exactly-once execution across all threads (uses internal mutex/flag) |
| Can you use `std::shared_mutex` in ISR? | No. No blocking primitives in interrupt context. |
