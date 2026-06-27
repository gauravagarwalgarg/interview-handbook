# Design Pattern Scenarios  Real-World Embedded Exercises

**Focus: Production-grade pattern implementations judgeable in a 1-hour interview**

---

## 🏗️ Observer Pattern  Multi-Subscriber Sensor Event System

### Scenario: Avionics Health Monitoring Bus

You are designing a health monitoring system for an aircraft. Multiple subsystems
need to react when any sensor reports an anomaly:

- **Flight Recorder**  logs every anomaly to black box
- **Pilot Display**  alerts crew for critical anomalies
- **Auto-Shutdown**  disables subsystem if threshold exceeded
- **Telemetry Downlink**  streams anomalies to ground station

**Constraints:**
- No heap allocation after initialization
- Maximum 16 observers (known at compile time)
- ISR-safe notification (some sensors trigger from interrupt context)
- Observer removal must not invalidate iteration
- Must handle observer priority (critical handlers before logging)

**Ask the candidate to design the complete system.**

### Expected Implementation

```cpp
#include <array>
#include <cstdint>
#include <atomic>
#include <algorithm>

// Event types
enum class AnomalyType : uint8_t {
    OverTemp, UnderVoltage, SensorFailure, CommLoss, Vibration
};

enum class Severity : uint8_t { Info, Warning, Critical, Emergency };

struct AnomalyEvent {
    AnomalyType type;
    Severity severity;
    uint32_t timestamp_ms;
    uint16_t source_id;
    float value;
    float threshold;
};
```


```cpp
// Observer interface  no virtual (ISR-safe, deterministic)
using AnomalyHandler = void(*)(const AnomalyEvent&, void* context);

struct ObserverSlot {
    AnomalyHandler handler = nullptr;
    void* context = nullptr;
    uint8_t priority = 255;         // Lower = higher priority
    AnomalyType filter = {};        // Which anomaly types to receive
    bool filter_all = true;         // Receive all types if true

    bool is_active() const { return handler != nullptr; }
};

template <std::size_t MaxObservers = 16>
class AnomalyBus {
public:
    struct SubscriptionHandle {
        uint8_t index = 0xFF;
        bool valid() const { return index != 0xFF; }
    };

    // Subscribe with priority (lower = notified first)
    SubscriptionHandle subscribe(AnomalyHandler handler, void* ctx,
                                  uint8_t priority = 128) {
        for (uint8_t i = 0; i < MaxObservers; ++i) {
            if (!slots_[i].is_active()) {
                slots_[i] = {handler, ctx, priority, {}, true};
                sorted_ = false;
                ++count_;
                return {i};
            }
        }
        return {};  // Full
    }

    // Subscribe with type filter
    SubscriptionHandle subscribe_filtered(AnomalyHandler handler, void* ctx,
                                           AnomalyType type, uint8_t priority = 128) {
        auto handle = subscribe(handler, ctx, priority);
        if (handle.valid()) {
            slots_[handle.index].filter = type;
            slots_[handle.index].filter_all = false;
        }
        return handle;
    }
```


```cpp
    void unsubscribe(SubscriptionHandle handle) {
        if (handle.valid() && handle.index < MaxObservers) {
            slots_[handle.index] = {};
            --count_;
        }
    }

    // ISR-safe notification  no allocation, no blocking
    void notify(const AnomalyEvent& event) {
        if (!sorted_) sort_by_priority();

        for (const auto& slot : dispatch_order_) {
            if (!slot->is_active()) continue;
            if (!slot->filter_all && slot->filter != event.type) continue;
            slot->handler(event, slot->context);
        }
    }

    std::size_t observer_count() const { return count_; }

private:
    void sort_by_priority() {
        // Build dispatch order (sorted pointers to active slots)
        std::size_t n = 0;
        for (auto& slot : slots_) {
            if (slot.is_active()) dispatch_order_[n++] = &slot;
        }
        std::sort(dispatch_order_.begin(), dispatch_order_.begin() + n,
                  [](const ObserverSlot* a, const ObserverSlot* b) {
                      return a->priority < b->priority;
                  });
        // Null-terminate
        for (std::size_t i = n; i < MaxObservers; ++i)
            dispatch_order_[i] = nullptr;
        sorted_ = true;
    }

    std::array<ObserverSlot, MaxObservers> slots_{};
    std::array<ObserverSlot*, MaxObservers> dispatch_order_{};
    std::size_t count_ = 0;
    bool sorted_ = false;
};
```


### Usage Example

```cpp
// Concrete observers
void flight_recorder_handler(const AnomalyEvent& e, void* ctx) {
    auto* recorder = static_cast<FlightRecorder*>(ctx);
    recorder->log(e);
}

void auto_shutdown_handler(const AnomalyEvent& e, void* ctx) {
    if (e.severity >= Severity::Critical) {
        auto* subsys = static_cast<Subsystem*>(ctx);
        subsys->emergency_shutdown();
    }
}

// Registration at init time
AnomalyBus<16> bus;
bus.subscribe(auto_shutdown_handler, &engine_subsystem, 0);   // Highest priority
bus.subscribe(flight_recorder_handler, &recorder, 50);         // Medium
bus.subscribe_filtered(pilot_alert_handler, &display,
                       AnomalyType::OverTemp, 10);             // Only OverTemp
```

### Evaluation Criteria

| Criterion | Weak (1-2) | Adequate (3) | Strong (4-5) |
|-----------|-----------|-------------|--------------|
| No heap | Uses `std::vector` | Fixed array but copies | Zero-alloc, pointer-based dispatch |
| ISR safety | Uses `std::function` | Function pointers but blocking sort | Pre-sorted, no mutex in notify path |
| Priority | FIFO only | Priority field but unsorted | Sorted dispatch with lazy re-sort |
| Filtering | All observers see all events | Runtime if-check | Compile-time or per-slot filter |
| Unsubscribe safety | Invalidates iteration | Nulls slot but no re-sort | Deferred removal or null-skip |

### Follow-Up Probes

1. How do you make `notify()` re-entrant? (What if a handler subscribes another observer?)
2. What if an observer throws? (Catch per-handler, log, continue  or `noexcept` contract)
3. How do you add thread-safety without mutex? (SPSC queue of events, single notify thread)
4. How would you unit test this? (Mock handlers that record calls, verify order/filtering)

---

## 🏗️ Factory Pattern  Device Driver Instantiation

### Scenario: Multi-Protocol Sensor Hub

Your IoT gateway supports multiple sensor types connected via different buses:
- I2C temperature sensors (TMP117, BME280)
- SPI IMUs (ICM-42688, LSM6DSO)
- UART GPS modules (NEO-M9N, L86)
- Analog sensors via ADC

At runtime, the system reads a hardware configuration (from EEPROM or Device Tree)
and must instantiate the correct driver for each detected sensor.

**Requirements:**
- No RTTI (`-fno-rtti` build)
- Drivers are registered at compile time (static registration, no `dlopen`)
- Each driver conforms to a common `ISensor` interface
- Memory for drivers comes from a static pool (no heap)
- Adding a new sensor type requires only adding one file (open-closed principle)


### Expected Implementation

```cpp
#include <array>
#include <cstdint>
#include <cstring>
#include <new>       // placement new
#include <span>

// Common sensor interface
class ISensor {
public:
    virtual ~ISensor() = default;
    virtual bool init() = 0;
    virtual bool read(std::span<uint8_t> buffer) = 0;
    virtual void reset() = 0;
    virtual const char* name() const = 0;
    virtual uint32_t sample_rate_hz() const = 0;
};

// Hardware descriptor (from EEPROM / Device Tree)
struct SensorDescriptor {
    char id[16];            // e.g., "TMP117", "ICM42688"
    uint8_t bus_type;       // 0=I2C, 1=SPI, 2=UART, 3=ADC
    uint8_t bus_index;
    uint8_t address;        // I2C addr or SPI CS
    uint32_t bus_speed_hz;
};

// Factory function signature
using SensorCreateFn = ISensor*(*)(void* memory, const SensorDescriptor& desc);
using SensorSizeFn = std::size_t(*)();

// Registration entry
struct SensorRegistryEntry {
    const char* sensor_id;
    SensorCreateFn create;
    SensorSizeFn size;
};
```

```cpp
// Static registry with auto-registration via global constructors
class SensorRegistry {
public:
    static constexpr std::size_t MaxDrivers = 32;

    static bool register_driver(const char* id, SensorCreateFn create,
                                 SensorSizeFn size) {
        if (count_ >= MaxDrivers) return false;
        entries_[count_++] = {id, create, size};
        return true;
    }

    static const SensorRegistryEntry* find(const char* id) {
        for (std::size_t i = 0; i < count_; ++i) {
            if (std::strcmp(entries_[i].sensor_id, id) == 0) {
                return &entries_[i];
            }
        }
        return nullptr;
    }

    static std::size_t max_driver_size() {
        std::size_t max = 0;
        for (std::size_t i = 0; i < count_; ++i) {
            max = std::max(max, entries_[i].size());
        }
        return max;
    }

private:
    static inline std::array<SensorRegistryEntry, MaxDrivers> entries_{};
    static inline std::size_t count_ = 0;
};
```


```cpp
// Auto-registration macro  adding a new sensor is ONE line
#define REGISTER_SENSOR(ClassName, SensorId) \
    static ISensor* create_##ClassName(void* mem, const SensorDescriptor& desc) { \
        return new (mem) ClassName(desc); \
    } \
    static std::size_t size_##ClassName() { return sizeof(ClassName); } \
    static bool reg_##ClassName = SensorRegistry::register_driver( \
        SensorId, create_##ClassName, size_##ClassName);

// Concrete driver  one file per sensor
class TMP117Driver : public ISensor {
public:
    explicit TMP117Driver(const SensorDescriptor& desc)
        : bus_index_(desc.bus_index), address_(desc.address) {}

    bool init() override {
        // Configure I2C, verify WHO_AM_I register
        return i2c_read_reg(bus_index_, address_, 0x0F) == 0x0117;
    }

    bool read(std::span<uint8_t> buffer) override {
        if (buffer.size() < sizeof(float)) return false;
        int16_t raw = i2c_read_reg16(bus_index_, address_, 0x00);
        float temp = raw * 0.0078125f;
        std::memcpy(buffer.data(), &temp, sizeof(float));
        return true;
    }

    void reset() override { i2c_write_reg(bus_index_, address_, 0x01, 0x0006); }
    const char* name() const override { return "TMP117"; }
    uint32_t sample_rate_hz() const override { return 4; }

private:
    uint8_t bus_index_;
    uint8_t address_;
};

REGISTER_SENSOR(TMP117Driver, "TMP117")  // <-- One line to add a new sensor!
```

```cpp
// Memory pool for sensor instances
class SensorPool {
public:
    static constexpr std::size_t MaxSensors = 8;
    static constexpr std::size_t SlotSize = 256;  // >= max driver size

    ISensor* create(const SensorDescriptor& desc) {
        auto* entry = SensorRegistry::find(desc.id);
        if (!entry || next_slot_ >= MaxSensors) return nullptr;
        if (entry->size() > SlotSize) return nullptr;

        void* mem = &pool_[next_slot_];
        ISensor* sensor = entry->create(mem, desc);
        ++next_slot_;
        return sensor;
    }

    void destroy_all() {
        for (std::size_t i = 0; i < next_slot_; ++i) {
            auto* sensor = reinterpret_cast<ISensor*>(&pool_[i]);
            sensor->~ISensor();
        }
        next_slot_ = 0;
    }

private:
    alignas(ISensor) std::array<
        std::aligned_storage_t<SlotSize, alignof(std::max_align_t)>,
        MaxSensors
    > pool_{};
    std::size_t next_slot_ = 0;
};
```


### Evaluation Criteria

| Criterion | Weak (1-2) | Adequate (3) | Strong (4-5) |
|-----------|-----------|-------------|--------------|
| Open-Closed | Modifying factory switch-case | String map + factory functions | Auto-registration, one file per driver |
| Memory | `new` / heap | Pre-sized `unique_ptr` array | Placement new in static pool |
| Extensibility | Hard-coded sensor list | Config-driven but manual | Registry pattern, zero-touch add |
| Type safety | `void*` everywhere | Base pointer with manual cast | Typed creation with size validation |
| No RTTI | Uses `dynamic_cast` | String-based type ID | Compile-time registration, no RTTI |

### Follow-Up Probes

1. How do you handle sensor hot-plug (USB sensor added at runtime)?
2. What if two drivers claim the same ID? (First-wins or assert at startup?)
3. How do you version the driver interface for backward compatibility?
4. How would you test this without hardware? (Mock `i2c_read_reg` at link time)

---

## 🏗️ State Pattern  Communication Protocol Parser

### Scenario: MAVLink Protocol State Machine

You're implementing a MAVLink v2 message parser for a drone flight controller.
The protocol has this byte-level structure:

```
┌─────┬──────┬─────┬────┬────┬──────┬──────────┬─────────┬────────┐
│ STX │ LEN  │ SEQ │SYS │CMP│MSG_ID│ PAYLOAD  │ CRC_LOW │CRC_HIGH│
│0xFD │1 byte│1    │1   │1  │3 byte│ LEN bytes│         │        │
└─────┴──────┴─────┴────┬────┴──────┴──────────┴─────────┴────────┘
```

**Requirements:**
- Byte-by-byte parsing (data arrives one byte at a time from UART ISR)
- Zero-copy (parse in-place from DMA ring buffer)
- Report statistics: valid frames, CRC errors, sync losses
- Handle corrupted streams gracefully (re-sync after garbage)
- Must process in < 1 μs per byte (ISR budget)


### Expected Implementation (enum-based state machine)

```cpp
#include <cstdint>
#include <array>
#include <cstring>

struct MavlinkMessage {
    uint8_t len;
    uint8_t seq;
    uint8_t sys_id;
    uint8_t comp_id;
    uint32_t msg_id;  // 24-bit, stored in 32
    std::array<uint8_t, 255> payload;
    uint16_t crc;
};

struct ParserStats {
    uint32_t valid_frames = 0;
    uint32_t crc_errors = 0;
    uint32_t sync_losses = 0;
    uint32_t bytes_processed = 0;
};

class MavlinkParser {
public:
    enum class State : uint8_t {
        WaitSync,
        GotLen,
        GotSeq,
        GotSysId,
        GotCompId,
        GotMsgId1,
        GotMsgId2,
        GotMsgId3,
        ReadingPayload,
        GotCrcLow,
        Complete
    };

    using FrameCallback = void(*)(const MavlinkMessage&, void*);

    void set_callback(FrameCallback cb, void* ctx) {
        callback_ = cb;
        ctx_ = ctx;
    }

    // Called from ISR  must be fast and non-blocking
    void feed_byte(uint8_t byte) {
        stats_.bytes_processed++;

        switch (state_) {
        case State::WaitSync:
            if (byte == 0xFD) {
                state_ = State::GotLen;
            } else {
                stats_.sync_losses++;
            }
            break;

        case State::GotLen:
            msg_.len = byte;
            payload_index_ = 0;
            crc_accumulator_ = 0xFFFF;
            crc_update(byte);
            state_ = State::GotSeq;
            break;

        case State::GotSeq:
            msg_.seq = byte;
            crc_update(byte);
            state_ = State::GotSysId;
            break;

        case State::GotSysId:
            msg_.sys_id = byte;
            crc_update(byte);
            state_ = State::GotCompId;
            break;

        case State::GotCompId:
            msg_.comp_id = byte;
            crc_update(byte);
            state_ = State::GotMsgId1;
            break;

        case State::GotMsgId1:
            msg_.msg_id = byte;
            crc_update(byte);
            state_ = State::GotMsgId2;
            break;

        case State::GotMsgId2:
            msg_.msg_id |= static_cast<uint32_t>(byte) << 8;
            crc_update(byte);
            state_ = State::GotMsgId3;
            break;

        case State::GotMsgId3:
            msg_.msg_id |= static_cast<uint32_t>(byte) << 16;
            crc_update(byte);
            if (msg_.len == 0) {
                state_ = State::GotCrcLow;
            } else {
                state_ = State::ReadingPayload;
            }
            break;

        case State::ReadingPayload:
            msg_.payload[payload_index_++] = byte;
            crc_update(byte);
            if (payload_index_ >= msg_.len) {
                state_ = State::GotCrcLow;
            }
            break;

        case State::GotCrcLow:
            msg_.crc = byte;
            state_ = State::Complete;
            break;

        case State::Complete:
            msg_.crc |= static_cast<uint16_t>(byte) << 8;
            if (msg_.crc == crc_accumulator_) {
                stats_.valid_frames++;
                if (callback_) callback_(msg_, ctx_);
            } else {
                stats_.crc_errors++;
            }
            state_ = State::WaitSync;  // Reset for next frame
            break;
        }
    }
```


```cpp
    void reset() { state_ = State::WaitSync; }
    const ParserStats& stats() const { return stats_; }

private:
    void crc_update(uint8_t byte) {
        uint8_t tmp = byte ^ (crc_accumulator_ & 0xFF);
        tmp ^= (tmp << 4);
        crc_accumulator_ = (crc_accumulator_ >> 8)
            ^ (static_cast<uint16_t>(tmp) << 8)
            ^ (static_cast<uint16_t>(tmp) << 3)
            ^ (tmp >> 4);
    }

    State state_ = State::WaitSync;
    MavlinkMessage msg_{};
    uint8_t payload_index_ = 0;
    uint16_t crc_accumulator_ = 0xFFFF;
    ParserStats stats_{};
    FrameCallback callback_ = nullptr;
    void* ctx_ = nullptr;
};
```

### Alternative: `std::variant`-Based State Machine (Discussion)

For more complex protocols with state-specific data:

```cpp
struct WaitSync {};
struct ReadHeader { uint8_t bytes_read = 0; };
struct ReadPayload { uint8_t remaining; };
struct ValidateCrc { uint8_t crc_low; };

using ParserState = std::variant<WaitSync, ReadHeader, ReadPayload, ValidateCrc>;

// Transition via std::visit
ParserState transition(WaitSync, uint8_t byte) {
    return (byte == 0xFD) ? ParserState{ReadHeader{}} : ParserState{WaitSync{}};
}
```

**When to use which:**

| Approach | Best For | Overhead |
|----------|----------|----------|
| Enum + switch | Simple linear protocols, ISR code | Minimal  single byte for state |
| `std::variant` + visit | Complex protocols with state-specific data | sizeof(largest state) + index byte |
| State pattern (virtual) | Plugin architectures, testable states | vtable pointer per state object |
| Coroutines (C++20) | Readable sequential parsing logic | Compiler-dependent, frame allocation |

### Evaluation Criteria

| Criterion | Weak (1-2) | Adequate (3) | Strong (4-5) |
|-----------|-----------|-------------|--------------|
| ISR timing | Allocates per byte | Simple switch but no CRC streaming | O(1) per byte, streaming CRC |
| Re-sync | Crashes on garbage | Resets to start | Scans for next STX, counts losses |
| Error handling | Ignores CRC | Checks CRC at end | Per-field validation, stats tracking |
| Testability | Only works with real UART | Accepts byte array | feed_byte() is pure, callback-based |
| Code structure | Giant function | Separate functions per state | Clean state machine pattern |

### Follow-Up Probes

1. How do you handle back-to-back frames with no gap? (Parser auto-resets to WaitSync after Complete)
2. What if the payload length field is corrupted (says 255 bytes)? (Timeout, or max-length check before entering ReadPayload)
3. How do you test this? (Feed known-good and known-bad byte sequences, verify stats)
4. How would you use C++20 coroutines to simplify this? (`co_await` each byte, sequential flow)

---

## 🏗️ Factory + State + Observer Combined  Device Lifecycle Manager

### Scenario: IoT Device Fleet Management

Your IoT platform manages device lifecycle with these states:


```
┌──────────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐
│Provisioned│────▶│Configuring│────▶│ Operational│────▶│ Updating │
└──────────┘     └──────────┘     └───────────┘     └──────────┘
      │                │                 │                  │
      │                │                 │                  │
      ▼                ▼                 ▼                  ▼
┌──────────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐
│  Error   │◀────│  Error   │◀────│   Error   │◀────│  Error   │
└──────────┘     └──────────┘     └───────────┘     └──────────┘
      │                                                     │
      └─────────────────────────────────────────────────────┘
                          Recovery
```

**Combines three patterns:**
- **Factory:** Create device handlers based on hardware revision
- **State:** Manage lifecycle transitions with validation
- **Observer:** Notify fleet management dashboard, logging, and alerting

**Ask candidate to:**
1. Define the state types with entry/exit actions
2. Design valid transitions (reject invalid ones)
3. Add observer notifications on state changes
4. Show how factory creates the correct device handler variant

### Expected Implementation (Sketch)

```cpp
// States with associated data
struct Provisioned { std::string device_id; std::string firmware_ver; };
struct Configuring { uint8_t progress_pct = 0; };
struct Operational { std::chrono::steady_clock::time_point since; uint32_t uptime_sec = 0; };
struct Updating { std::string target_version; uint8_t progress_pct = 0; };
struct Error { uint32_t code; std::string message; std::string previous_state; };

using DeviceState = std::variant<Provisioned, Configuring, Operational, Updating, Error>;

// Events
struct ConfigComplete {};
struct ConfigFailed { uint32_t error_code; };
struct UpdateAvailable { std::string version; };
struct UpdateComplete {};
struct UpdateFailed { uint32_t error_code; };
struct FaultDetected { uint32_t code; std::string msg; };
struct RecoveryAttempt {};

// State machine with observer notifications
class DeviceLifecycle {
public:
    using StateChangeHandler = void(*)(const DeviceState& from,
                                        const DeviceState& to, void* ctx);

    void on_state_change(StateChangeHandler handler, void* ctx) {
        observers_[obs_count_++] = {handler, ctx};
    }

    template <typename Event>
    bool handle(const Event& event) {
        auto new_state = std::visit(
            [&event](auto& current) -> std::optional<DeviceState> {
                return try_transition(current, event);
            },
            state_);

        if (new_state) {
            auto old = state_;
            state_ = *new_state;
            notify_observers(old, state_);
            return true;
        }
        return false;  // Invalid transition  rejected
    }

private:
    // Valid transitions (compile-time enforced via overloads)
    static std::optional<DeviceState> try_transition(Provisioned&, const ConfigComplete&) {
        return Configuring{};
    }
    static std::optional<DeviceState> try_transition(Configuring&, const ConfigComplete&) {
        return Operational{std::chrono::steady_clock::now()};
    }
    static std::optional<DeviceState> try_transition(Operational&, const UpdateAvailable& u) {
        return Updating{u.version};
    }
    static std::optional<DeviceState> try_transition(Updating&, const UpdateComplete&) {
        return Operational{std::chrono::steady_clock::now()};
    }
    // Error from any state
    template <typename State>
    static std::optional<DeviceState> try_transition(State&, const FaultDetected& f) {
        return Error{f.code, f.msg, typeid(State).name()};
    }
    // Default: invalid transition
    template <typename State, typename Event>
    static std::optional<DeviceState> try_transition(State&, const Event&) {
        return std::nullopt;
    }

    void notify_observers(const DeviceState& from, const DeviceState& to) {
        for (std::size_t i = 0; i < obs_count_; ++i) {
            observers_[i].handler(from, to, observers_[i].ctx);
        }
    }

    DeviceState state_ = Provisioned{};
    struct Observer { StateChangeHandler handler; void* ctx; };
    std::array<Observer, 8> observers_{};
    std::size_t obs_count_ = 0;
};
```


### Evaluation Criteria

| Criterion | Weak (1-2) | Adequate (3) | Strong (4-5) |
|-----------|-----------|-------------|--------------|
| State safety | Giant if/else, can enter invalid states | enum state, runtime checks | Variant-based, invalid transitions rejected |
| Pattern integration | Patterns isolated, don't compose | Basic composition | Seamless: factory creates, state manages, observers react |
| Error recovery | Crashes or ignores errors | Error state exists but no recovery | Error captures context, recovery path defined |
| Testability | Needs full system running | Can test state machine in isolation | Pure transition functions, mockable observers |

---

## 🏗️ Additional Scenario Templates (Interviewer Pick-List)

### Template A: Singleton  Hardware Peripheral Access

**Scenario:** UART peripheral driver that must be accessed from multiple threads but only initialized once.

**Key discussion points:**
- Meyer's Singleton (`static` local)  thread-safe since C++11
- Why Singleton is often an anti-pattern (testing difficulty, hidden dependencies)
- Better alternative: explicit dependency injection with reference
- When Singleton IS appropriate: hardware peripherals with exactly one instance

```cpp
class UartPort {
public:
    static UartPort& instance() {
        static UartPort port;  // Thread-safe, constructed once
        return port;
    }

    // Delete copy/move
    UartPort(const UartPort&) = delete;
    UartPort& operator=(const UartPort&) = delete;

private:
    UartPort() { /* init hardware */ }
    ~UartPort() { /* deinit hardware */ }
};
```

**Better pattern for testability:**
```cpp
// Dependency injection  pass reference, don't use global
class TelemetryService {
public:
    explicit TelemetryService(IUart& uart) : uart_(uart) {}
    // Testable with mock IUart!
private:
    IUart& uart_;
};
```

---

### Template B: Builder Pattern  Configuration Objects

**Scenario:** Build a complex network configuration for an IoT device:

```cpp
class NetworkConfig {
public:
    class Builder {
    public:
        Builder& ssid(std::string_view s) { cfg_.ssid_ = s; return *this; }
        Builder& password(std::string_view p) { cfg_.password_ = p; return *this; }
        Builder& static_ip(uint32_t ip) { cfg_.use_dhcp_ = false; cfg_.ip_ = ip; return *this; }
        Builder& dns(uint32_t primary, uint32_t secondary = 0) {
            cfg_.dns_primary_ = primary;
            cfg_.dns_secondary_ = secondary;
            return *this;
        }
        Builder& timeout_ms(uint32_t ms) { cfg_.timeout_ms_ = ms; return *this; }
        Builder& retries(uint8_t n) { cfg_.retries_ = n; return *this; }

        NetworkConfig build() const {
            // Validate before returning
            if (cfg_.ssid_.empty()) { /* error */ }
            return cfg_;
        }
    private:
        NetworkConfig cfg_;
    };

private:
    // All fields have sane defaults
    std::string ssid_;
    std::string password_;
    bool use_dhcp_ = true;
    uint32_t ip_ = 0;
    uint32_t dns_primary_ = 0;
    uint32_t dns_secondary_ = 0;
    uint32_t timeout_ms_ = 5000;
    uint8_t retries_ = 3;

    friend class Builder;
};

// Usage
auto config = NetworkConfig::Builder()
    .ssid("FieldNetwork")
    .password("secure123")
    .timeout_ms(10000)
    .retries(5)
    .build();
```

---

### Template C: Adapter Pattern  Legacy C API Wrapping

**Scenario:** Wrap a vendor's C-only sensor SDK into a modern C++ interface:

```cpp
// Vendor's C API (cannot modify)
extern "C" {
    typedef void* sensor_handle_t;
    int sensor_open(sensor_handle_t* handle, int bus, int addr);
    int sensor_read(sensor_handle_t handle, float* data, int count);
    int sensor_close(sensor_handle_t handle);
}

// Modern C++ adapter
class SensorAdapter {
public:
    SensorAdapter(int bus, int addr) {
        if (sensor_open(&handle_, bus, addr) != 0) {
            throw std::runtime_error("Failed to open sensor");  // Or return error
        }
    }

    ~SensorAdapter() noexcept {
        if (handle_) sensor_close(handle_);
    }

    // Move-only (transfer handle ownership)
    SensorAdapter(SensorAdapter&& other) noexcept
        : handle_(std::exchange(other.handle_, nullptr)) {}

    SensorAdapter& operator=(SensorAdapter&& other) noexcept {
        if (this != &other) {
            if (handle_) sensor_close(handle_);
            handle_ = std::exchange(other.handle_, nullptr);
        }
        return *this;
    }

    SensorAdapter(const SensorAdapter&) = delete;
    SensorAdapter& operator=(const SensorAdapter&) = delete;

    std::optional<std::vector<float>> read(int count) {
        std::vector<float> data(count);
        if (sensor_read(handle_, data.data(), count) == 0) {
            return data;
        }
        return std::nullopt;
    }

private:
    sensor_handle_t handle_ = nullptr;
};
```

---

## Pattern Selection Guide for Interviewers

Choose based on candidate's experience and target level:

| Pattern | Difficulty | Time Needed | Best For Level |
|---------|-----------|-------------|----------------|
| Observer (Anomaly Bus) | Medium-Hard | 20 min | Senior |
| Factory (Sensor Hub) | Hard | 20 min | Senior / Staff |
| State Machine (Protocol Parser) | Medium | 15 min | Mid / Senior |
| Combined (Lifecycle Manager) | Hard | 25 min | Staff |
| Singleton (discussion only) | Easy | 5 min | Junior / Mid |
| Builder (Config) | Easy-Medium | 10 min | Mid |
| Adapter (C wrap) | Medium | 10 min | Mid / Senior |

**Interviewer tip:** Start with an easier pattern to build confidence, then push into the harder combined scenario if time permits.
