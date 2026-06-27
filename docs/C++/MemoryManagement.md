# Memory Management  Deep Dive

**Focus: Smart Pointers, RAII, Custom Allocators, Memory Layout, Embedded Constraints**

---

## 🔬 Q1: RAII in Practice  Resource Wrapper Design

**Question:** You are writing firmware for a sensor hub that manages multiple SPI peripherals. Design a RAII wrapper for an SPI bus lock that:
1. Acquires the SPI bus mutex on construction
2. Configures clock polarity/phase
3. Releases the bus on destruction (even if an exception occurs during transfer)

**What the interviewer evaluates:**
- Understanding of deterministic destruction
- Exception safety guarantees (basic vs. strong)
- Awareness of `noexcept` on destructors
- Whether candidate considers move semantics (non-copyable, movable?)

**Expected answer structure:**

```cpp
class SpiBusLock {
public:
    explicit SpiBusLock(SpiBus& bus, SpiConfig config)
        : bus_(bus) {
        bus_.acquire();           // Lock mutex
        bus_.configure(config);   // Set clock/phase
    }

    ~SpiBusLock() noexcept {
        bus_.release();           // Always release, even on exception
    }

    // Non-copyable, non-movable (bus lock shouldn't transfer)
    SpiBusLock(const SpiBusLock&) = delete;
    SpiBusLock& operator=(const SpiBusLock&) = delete;
    SpiBusLock(SpiBusLock&&) = delete;
    SpiBusLock& operator=(SpiBusLock&&) = delete;

    void transfer(std::span<const uint8_t> tx, std::span<uint8_t> rx) {
        bus_.transact(tx, rx);
    }

private:
    SpiBus& bus_;
};
```

**Follow-up probes:**
- Why `noexcept` on the destructor?
- What if we want to support move semantics (e.g., returning the lock from a factory)?
- How would you handle a failed `acquire()`  constructor throws?

---

## 🔬 Q2: Smart Pointer Ownership in an Event System

**Question:** You are building a publish-subscribe event system for an embedded Linux application. Sensors publish events; multiple handlers subscribe.

Design the ownership model:
- Who owns the event object?
- What pointer types do you use for publisher, subscriber, and the event bus?
- How do you handle subscriber lifetime (subscriber may be destroyed before events arrive)?

**Expected discussion points:**

| Component | Ownership | Pointer Type |
|-----------|-----------|-------------|
| Event Bus | Owns subscriptions | `std::vector<std::weak_ptr<Handler>>` |
| Publisher | Does not own handlers | Passes events by value or `shared_ptr<const Event>` |
| Subscriber | Owns itself or shared with bus | `std::shared_ptr<Handler>` (caller holds) |

```cpp
class EventBus {
public:
    void subscribe(std::weak_ptr<IHandler> handler) {
        handlers_.push_back(std::move(handler));
    }

    void publish(const Event& event) {
        handlers_.erase(
            std::remove_if(handlers_.begin(), handlers_.end(),
                [&event](auto& wptr) {
                    if (auto sptr = wptr.lock()) {
                        sptr->handle(event);
                        return false;
                    }
                    return true;  // Expired  remove
                }),
            handlers_.end());
    }

private:
    std::vector<std::weak_ptr<IHandler>> handlers_;
};
```

**Follow-up probes:**
- What's the cost of `shared_ptr` in an ISR context? (Answer: unacceptable  reference counting is not lock-free on all platforms)
- How would you redesign for a bare-metal RTOS? (Answer: static subscriber table, raw function pointers or `std::function` with fixed-size storage)

---

## 🔬 Q3: Custom Allocator for Real-Time Systems

**Question:** Your team needs a memory pool allocator for a flight controller that:
- Allocates fixed-size blocks (no fragmentation)
- O(1) allocation and deallocation
- No use of `malloc`/`free` at runtime
- Thread-safe for two threads (main loop + DMA ISR)

Sketch the allocator interface compatible with STL containers.

**Key evaluation points:**
- Free-list (intrusive linked list within the pool)
- Static memory block (`alignas` + `std::array` or `std::byte[]`)
- Lock-free approach for ISR context (single-producer-single-consumer with atomic index)
- STL Allocator concept compliance (`allocate()`, `deallocate()`, `value_type`)

```cpp
template <typename T, std::size_t PoolSize>
class FixedPoolAllocator {
public:
    using value_type = T;

    FixedPoolAllocator() noexcept {
        // Initialize free list
        for (std::size_t i = 0; i < PoolSize - 1; ++i) {
            reinterpret_cast<Node*>(&pool_[i])->next = 
                reinterpret_cast<Node*>(&pool_[i + 1]);
        }
        reinterpret_cast<Node*>(&pool_[PoolSize - 1])->next = nullptr;
        free_head_ = reinterpret_cast<Node*>(&pool_[0]);
    }

    T* allocate(std::size_t n) {
        if (n != 1 || !free_head_) throw std::bad_alloc();
        Node* block = free_head_;
        free_head_ = free_head_->next;
        return reinterpret_cast<T*>(block);
    }

    void deallocate(T* p, std::size_t) noexcept {
        auto* node = reinterpret_cast<Node*>(p);
        node->next = free_head_;
        free_head_ = node;
    }

private:
    struct Node { Node* next; };
    alignas(T) std::byte pool_[PoolSize][sizeof(T)];
    Node* free_head_ = nullptr;
};
```

**Follow-up probes:**
- How do you make this ISR-safe? (atomic compare-exchange on `free_head_`)
- What about alignment requirements for DMA buffers? (`alignas(32)` or cache-line alignment)
- Can you use `std::pmr::memory_resource` (C++17) instead?

---

## 🔬 Q4: Memory-Mapped I/O and Volatile

**Question:** On an ARM Cortex-M4, you have a peripheral register at address `0x4000'1000`. The register is:
- Read to get status
- Written to clear flags

Write a type-safe C++ abstraction for this register. Explain why `volatile` is necessary and what it does NOT guarantee.

**Expected answer:**

```cpp
template <typename T, std::uintptr_t Address>
class MmioRegister {
public:
    static_assert(std::is_trivially_copyable_v<T>);

    void write(T value) noexcept {
        *reinterpret_cast<volatile T*>(Address) = value;
    }

    T read() const noexcept {
        return *reinterpret_cast<volatile T*>(Address);
    }

    void set_bits(T mask) noexcept {
        write(read() | mask);
    }

    void clear_bits(T mask) noexcept {
        write(read() & ~mask);
    }
};

// Usage
using StatusReg = MmioRegister<uint32_t, 0x4000'1000>;
StatusReg status;
auto flags = status.read();
```

**Key discussion:**
- `volatile` prevents compiler reordering/optimization of accesses but does NOT provide:
  - Atomicity (need atomic types or disable interrupts)
  - Memory ordering guarantees across cores (need barriers: `__DMB()`, `std::atomic_thread_fence`)
  - Cache coherency (hardware-dependent, often registers are in non-cacheable address space)

---

## 🔬 Q5: Stack vs Heap  Firmware Decision Matrix

**Question:** In a safety-critical firmware project (DO-178C / MISRA-compliant), the team has a strict "no heap allocation after initialization" policy. How do you:
1. Use STL containers without heap allocation?
2. Handle variable-length messages from a CAN bus?
3. Manage object polymorphism without `new`?

**Expected answers:**

1. **Static containers:** Use `std::array`, `etl::vector` (Embedded Template Library), or `std::pmr` with a `monotonic_buffer_resource` initialized at startup.

2. **Ring buffers with max-size:** Pre-allocate worst-case buffers. Use `std::variant` or tagged unions for variable-length payloads.

3. **Static polymorphism:** CRTP (Curiously Recurring Template Pattern) eliminates vtable overhead. For runtime polymorphism without heap: placement new into pre-allocated aligned storage, or `std::variant` with `std::visit`.

```cpp
// Static polymorphism via CRTP
template <typename Derived>
class SensorBase {
public:
    void sample() {
        static_cast<Derived*>(this)->sample_impl();
    }
};

class Accelerometer : public SensorBase<Accelerometer> {
public:
    void sample_impl() { /* read SPI */ }
};
```
