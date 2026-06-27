# C++ in Constrained Environments

**Focus: No-Heap, Deterministic, Real-Time, Safety-Critical C++**

---

## 🔬 Q1: MISRA C++ and Forbidden Features

**Question:** You're working on DO-178C DAL-A avionics software. Which C++ features are typically forbidden or restricted, and why?

**Expected answer:**

| Feature | Status | Reason |
|---------|--------|--------|
| Dynamic allocation (`new`/`delete`) | ❌ Forbidden after init | Non-deterministic timing, fragmentation |
| Exceptions (`throw`/`catch`) | ❌ Forbidden | Non-deterministic stack unwinding, code size |
| RTTI (`dynamic_cast`, `typeid`) | ❌ Forbidden | Runtime overhead, code bloat |
| `std::function` | ⚠️ Restricted | May heap-allocate for large callables |
| Virtual functions | ⚠️ Restricted | Indirect call cost, harder to analyze WCET |
| Templates | ✅ Allowed (with limits) | Code bloat risk from instantiation, but zero runtime cost |
| `constexpr` | ✅ Encouraged | Moves computation to compile time |
| Smart pointers | ⚠️ `unique_ptr` only | `shared_ptr` uses heap for control block |

**Follow-up:** How do you write polymorphic code without virtual functions?
- CRTP (Curiously Recurring Template Pattern)
- `std::variant` + `std::visit`
- Function pointer tables (manual vtable)
- Template-based policy design

---

## 🔬 Q2: Worst-Case Execution Time (WCET) Analysis

**Question:** How does C++ feature choice affect WCET analysis? Give examples of deterministic vs. non-deterministic constructs.

**Deterministic (WCET-friendly):**
- Stack-allocated objects with trivial destructors
- `constexpr` computations
- Array indexing (vs. linked list traversal)
- Inline functions
- Statically-bound function calls

**Non-deterministic (WCET-hostile):**
- `malloc`/`new`  depends on heap state, fragmentation
- `std::map`/`std::unordered_map`  rebalancing, rehashing
- Exception unwinding  path-dependent stack traversal
- `std::shared_ptr`  atomic ref count contention
- `std::string`  SSO threshold varies, may heap-allocate

---

## 🔬 Q3: Error Handling Without Exceptions

**Question:** Design an error-handling strategy for a firmware project that cannot use exceptions.

**Expected approaches (in order of preference):**

```cpp
// 1. std::expected (C++23) or equivalent
template <typename T, typename E = ErrorCode>
using Result = std::expected<T, E>;

Result<SensorData> read_sensor() {
    if (!is_ready()) return std::unexpected(ErrorCode::NotReady);
    return SensorData{/* ... */};
}

// 2. Return-code + out-parameter (C-style, but type-safe)
enum class Status : uint8_t { Ok, Timeout, HwFault, InvalidParam };

Status read_temperature(float& out_temp) {
    if (!sensor_powered_) return Status::HwFault;
    out_temp = adc_to_celsius(read_adc());
    return Status::Ok;
}

// 3. std::optional for "might not have a value"
std::optional<GpsCoordinate> get_position() {
    if (!gps_has_fix()) return std::nullopt;
    return parse_nmea();
}

// 4. Callback-based error propagation
using ErrorHandler = void(*)(ErrorCode, const char* context);
void set_error_handler(ErrorHandler h);
```

**Key design decisions:**
- `[[nodiscard]]` on all functions returning error codes
- Central error handler for fatal/unrecoverable errors (reboot, enter safe mode)
- Error codes as `enum class` (not raw ints) for type safety

---

## 🔬 Q4: Static Polymorphism with CRTP

**Question:** Implement a compile-time polymorphic interface for communication peripherals (UART, SPI, I2C) using CRTP:

```cpp
template <typename Derived>
class CommInterface {
public:
    bool init() {
        return static_cast<Derived*>(this)->init_impl();
    }

    int write(std::span<const uint8_t> data) {
        return static_cast<Derived*>(this)->write_impl(data);
    }

    int read(std::span<uint8_t> buffer, std::chrono::milliseconds timeout) {
        return static_cast<Derived*>(this)->read_impl(buffer, timeout);
    }

    void deinit() {
        static_cast<Derived*>(this)->deinit_impl();
    }
};

class UartDriver : public CommInterface<UartDriver> {
    friend class CommInterface<UartDriver>;

    bool init_impl() {
        // Configure baud rate, parity, etc.
        return configure_uart_peripheral();
    }

    int write_impl(std::span<const uint8_t> data) {
        return hal_uart_transmit(data.data(), data.size());
    }

    int read_impl(std::span<uint8_t> buffer, std::chrono::milliseconds timeout) {
        return hal_uart_receive(buffer.data(), buffer.size(), timeout.count());
    }

    void deinit_impl() {
        hal_uart_disable();
    }
};

// Usage  no vtable, fully inlined
template <typename T>
void send_telemetry(CommInterface<T>& comm, const TelemetryPacket& pkt) {
    auto bytes = serialize(pkt);
    comm.write(bytes);
}
```

**Comparison with virtual interface:**

| Aspect | CRTP | Virtual |
|--------|------|---------|
| Runtime cost | Zero (inlined) | Indirect call + cache miss |
| Binary size | May bloat (per instantiation) | Single implementation |
| WCET analysis | Deterministic | Harder (indirect jump) |
| Runtime flexibility | None (compile-time) | Can swap at runtime |
| Code complexity | Higher | Simpler |

---

## 🔬 Q5: Embedded Template Library (ETL) and Alternatives to STL

**Question:** The standard STL is often unsuitable for embedded. What alternatives exist and when do you use them?

| Library | Key Features | Use Case |
|---------|-------------|----------|
| ETL (Embedded Template Library) | Fixed-capacity containers, no heap, deterministic | Production firmware |
| EASTL | Game-engine optimized, custom allocators | High-performance embedded Linux |
| Boost.StaticVector | Fixed capacity, stack-allocated vector | When Boost is available |
| `std::pmr` (C++17) | Polymorphic allocators, monotonic buffers | Controlled allocation |
| Roll-your-own | Maximum control, audit-friendly | Safety-critical (DO-178C, ISO 26262) |

**Example using ETL:**

```cpp
#include <etl/vector.h>
#include <etl/string.h>
#include <etl/queue_spsc_atomic.h>

// Fixed-capacity vector  no heap, compile-time max
etl::vector<SensorReading, 64> readings;

// Fixed-capacity string  no SSO surprises
etl::string<128> log_message;

// Lock-free SPSC queue for ISR → task communication
etl::queue_spsc_atomic<Event, 32> event_queue;
```

---

## ⚡ Quick Constrained-Environment Checks

| Question | Key Answer |
|----------|-----------|
| What compiler flags disable exceptions? | `-fno-exceptions` (GCC/Clang), `/EHs-` (MSVC) |
| What is the `-fno-rtti` flag? | Disables RTTI, saves ~5-10% code size, breaks `dynamic_cast`/`typeid` |
| How do you prevent heap allocation in a translation unit? | Override `operator new` to `assert(false)` or link with no-malloc stubs |
| What is `-ffreestanding`? | No hosted standard library  only freestanding headers (`<cstdint>`, `<type_traits>`, etc.) |
| What is the "zero-overhead abstraction" principle? | You don't pay for what you don't use; what you use, you couldn't hand-code any better |
