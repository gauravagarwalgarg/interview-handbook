# OOP & Design Patterns  1-Hour Judgeable Exercises

**Focus: Patterns applicable to embedded/IoT/aerospace systems, constrained environments, production firmware**

---

## 🏗️ Exercise 1: State Machine Pattern  Flight Mode Controller (20 min)

**Scenario:** Design a state machine for a drone flight controller with these states:
- `Idle` → `Armed` → `Takeoff` → `Hovering` → `Landing` → `Idle`
- `Hovering` → `ReturnToHome` (on link loss)
- Any state → `Emergency` (on critical fault)

**Requirements:**
- Type-safe state transitions (invalid transitions should be compile errors or runtime-rejected)
- Each state has `on_enter()`, `on_exit()`, `on_event()` handlers
- No dynamic allocation (suitable for MCU deployment)
- Testable in isolation

**Ask the candidate to:**
1. Choose a pattern (State pattern, `std::variant` + `std::visit`, enum + switch)
2. Implement the core structure
3. Justify trade-offs

**Expected approach (variant-based state machine):**

```cpp
// States as types
struct Idle {};
struct Armed { std::chrono::steady_clock::time_point armed_at; };
struct Takeoff { float target_altitude; };
struct Hovering { float altitude; };
struct Landing {};
struct Emergency { uint32_t fault_code; };
struct ReturnToHome { Coordinate home_pos; };

// Events
struct ArmCommand {};
struct TakeoffCommand { float altitude; };
struct LinkLoss {};
struct CriticalFault { uint32_t code; };
struct Landed {};

using FlightState = std::variant<Idle, Armed, Takeoff, Hovering, Landing, Emergency, ReturnToHome>;

class FlightController {
public:
    template <typename Event>
    void handle(const Event& event) {
        state_ = std::visit(
            [&event](auto& current_state) -> FlightState {
                return transition(current_state, event);
            },
            state_);
    }

private:
    FlightState state_ = Idle{};

    // Valid transitions
    static FlightState transition(Idle&, const ArmCommand&) { return Armed{}; }
    static FlightState transition(Armed&, const TakeoffCommand& cmd) {
        return Takeoff{cmd.altitude};
    }
    // ... other valid transitions ...

    // Universal emergency transition
    template <typename State>
    static FlightState transition(State&, const CriticalFault& f) {
        return Emergency{f.code};
    }

    // Default: no transition (stay in current state)
    template <typename State, typename Event>
    static FlightState transition(State& s, const Event&) { return s; }
};
```

**Evaluation criteria:**
| Criterion | Weak (1-2) | Adequate (3) | Strong (4-5) |
|-----------|-----------|-------------|--------------|
| Pattern choice | Enum + giant switch | State pattern with virtual | Variant + visit or CRTP |
| Type safety | Raw integers for states | Enums but runtime checks | Compile-time invalid transition rejection |
| Memory | Heap-allocated state objects | Stack but wasteful | Variant (stack, sized to largest state) |
| Testability | Tightly coupled to HW | Interface-separated | Pure functions, injectable deps |

---

## 🏗️ Exercise 2: Observer Pattern  Sensor Health Monitoring (15 min)

**Scenario:** Multiple subsystems (telemetry, logging, safety monitor) need to be notified when a sensor reports an anomaly. Design the notification system.

**Constraints:**
- No heap allocation after init
- Maximum 8 observers (known at compile time)
- Must work on bare-metal (no exceptions, no RTTI)
- Observer removal must be safe (no dangling pointers)

**Expected approach:**

```cpp
template <typename EventT, std::size_t MaxObservers = 8>
class Subject {
public:
    using Callback = void(*)(const EventT&, void* context);

    struct Subscription {
        Callback cb = nullptr;
        void* context = nullptr;
    };

    bool subscribe(Callback cb, void* ctx = nullptr) {
        for (auto& slot : observers_) {
            if (!slot.cb) {
                slot = {cb, ctx};
                return true;
            }
        }
        return false;  // Full
    }

    void unsubscribe(Callback cb, void* ctx = nullptr) {
        for (auto& slot : observers_) {
            if (slot.cb == cb && slot.context == ctx) {
                slot = {};
                return;
            }
        }
    }

    void notify(const EventT& event) const {
        for (const auto& slot : observers_) {
            if (slot.cb) {
                slot.cb(event, slot.context);
            }
        }
    }

private:
    std::array<Subscription, MaxObservers> observers_{};
};
```

**Follow-up probes:**
- Why function pointers + context instead of `std::function`? (No heap, deterministic size, no exceptions)
- How would you make `notify()` re-entrant safe? (Copy observers array before iterating, or use a dirty flag)
- What's the alternative in C++20? (`std::function_ref` for non-owning callable references)

---

## 🏗️ Exercise 3: Command Pattern  Remote Command Executor (15 min)

**Scenario:** A ground station sends commands to a satellite. Each command must be:
- Validated before execution
- Executable with rollback capability (undo)
- Logged with timestamp and result
- Queued for deferred execution during communication windows

**Design the command framework.**

**Expected answer:**

```cpp
enum class CmdResult { Success, Failed, Rejected, Timeout };

class ICommand {
public:
    virtual ~ICommand() = default;
    virtual bool validate() const = 0;
    virtual CmdResult execute() = 0;
    virtual CmdResult undo() = 0;
    virtual std::string_view name() const = 0;
};

class CommandQueue {
public:
    void enqueue(std::unique_ptr<ICommand> cmd) {
        if (count_ < MaxCommands) {
            queue_[count_++] = std::move(cmd);
        }
    }

    void execute_all() {
        for (std::size_t i = 0; i < count_; ++i) {
            auto& cmd = queue_[i];
            if (cmd->validate()) {
                auto result = cmd->execute();
                log(cmd->name(), result);
                if (result == CmdResult::Success) {
                    history_[hist_count_++] = std::move(cmd);
                }
            } else {
                log(cmd->name(), CmdResult::Rejected);
            }
        }
        count_ = 0;
    }

    void undo_last() {
        if (hist_count_ > 0) {
            history_[--hist_count_]->undo();
        }
    }

private:
    static constexpr std::size_t MaxCommands = 32;
    std::array<std::unique_ptr<ICommand>, MaxCommands> queue_;
    std::array<std::unique_ptr<ICommand>, MaxCommands> history_;
    std::size_t count_ = 0;
    std::size_t hist_count_ = 0;

    void log(std::string_view name, CmdResult result) { /* ... */ }
};

// Concrete command example
class SetTransmitPowerCommand : public ICommand {
public:
    explicit SetTransmitPowerCommand(float new_power_dbm)
        : target_power_(new_power_dbm) {}

    bool validate() const override {
        return target_power_ >= -10.0f && target_power_ <= 30.0f;
    }

    CmdResult execute() override {
        previous_power_ = radio_.get_power();
        return radio_.set_power(target_power_) ? CmdResult::Success : CmdResult::Failed;
    }

    CmdResult undo() override {
        return radio_.set_power(previous_power_) ? CmdResult::Success : CmdResult::Failed;
    }

    std::string_view name() const override { return "SetTransmitPower"; }

private:
    float target_power_;
    float previous_power_ = 0.0f;
    Radio& radio_;  // Injected dependency
};
```

---

## 🏗️ Exercise 4: Strategy Pattern  Data Compression Selection (10 min)

**Scenario:** A data recorder on an autonomous vehicle must compress telemetry data before writing to flash. Different compression strategies apply based on available CPU budget and data type (IMU vs. video thumbnails vs. logs).

**Design the strategy selection.**

```cpp
class ICompressor {
public:
    virtual ~ICompressor() = default;
    virtual std::span<const uint8_t> compress(std::span<const uint8_t> input,
                                               std::span<uint8_t> output) = 0;
    virtual std::string_view algorithm_name() const = 0;
    virtual std::size_t worst_case_size(std::size_t input_size) const = 0;
};

class LZ4Compressor : public ICompressor { /* ... */ };
class RunLengthCompressor : public ICompressor { /* ... */ };
class NullCompressor : public ICompressor { /* passthrough */ };

class DataRecorder {
public:
    void set_strategy(ICompressor& compressor) {
        compressor_ = &compressor;
    }

    void record(std::span<const uint8_t> data) {
        auto compressed = compressor_->compress(data, work_buffer_);
        flash_.write(compressed);
    }

private:
    ICompressor* compressor_ = nullptr;  // Non-owning
    std::array<uint8_t, 4096> work_buffer_;
    FlashDriver& flash_;
};
```

**Key discussion:** Why pointer-to-interface (non-owning) instead of `std::unique_ptr`? Because strategies are typically longer-lived than the recorder, or statically allocated.

---

## Design Pattern Summary  Embedded Applicability

| Pattern | Embedded Use Case | Heap-Free? | Typical C++ Mechanism |
|---------|------------------|-----------|----------------------|
| State Machine | Flight modes, protocol parsers | ✅ | `std::variant` + `std::visit` |
| Observer | Sensor event notification | ✅ | Static array of function pointers |
| Command | Remote command execution, undo | ⚠️ (with pool) | Virtual interface + queue |
| Strategy | Algorithm selection at runtime | ✅ | Pointer-to-interface |
| Singleton | Hardware peripheral access | ✅ | `static` local with deleted copy |
| Factory | Object creation without `new` | ✅ (placement new) | Template + aligned storage |
| CRTP | Static polymorphism, mixin | ✅ | Template inheritance |
| Type Erasure | Callbacks, any-handler | ⚠️ | Small buffer optimization |

---

## ⚡ Quick OOP Checks

| Question | Key Answer |
|----------|-----------|
| What is the diamond problem? | Ambiguity from multiple inheritance of same base  resolve with virtual inheritance |
| What does `override` keyword do? | Ensures function actually overrides a virtual  compile error if signature mismatch |
| What is object slicing? | Assigning derived to base by value loses derived-specific data |
| When is a class "polymorphic"? | When it has at least one virtual function (enables `dynamic_cast`, `typeid`) |
| Cost of virtual function call? | Indirect call via vtable pointer  ~1-3 cycles extra + possible cache miss + prevents inlining |
| What is the Rule of Zero? | If your class doesn't manage a resource directly, don't define any special member functions |
