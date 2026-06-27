# Modern C++14/17/20  Firmware-Relevant Features

**Focus: Features that matter for embedded/IoT/aerospace production code**

---

## 🔬 Q1: `constexpr` and Compile-Time Computation

**Question:** You need a CRC-32 lookup table for a communication protocol. The table is 1 KB and never changes. Show how to compute it entirely at compile time using `constexpr`.

**Why this matters:** Embedded systems benefit from moving computation to compile time  no runtime cost, no flash initialization code, table lives in `.rodata` (ROM).

**Expected answer:**

```cpp
constexpr uint32_t crc32_for_byte(uint32_t byte) {
    constexpr uint32_t polynomial = 0xEDB88320;
    for (int i = 0; i < 8; ++i) {
        byte = (byte & 1) ? (byte >> 1) ^ polynomial : byte >> 1;
    }
    return byte;
}

constexpr auto generate_crc32_table() {
    std::array<uint32_t, 256> table{};
    for (uint32_t i = 0; i < 256; ++i) {
        table[i] = crc32_for_byte(i);
    }
    return table;
}

inline constexpr auto crc32_table = generate_crc32_table();

constexpr uint32_t crc32(std::span<const uint8_t> data) {
    uint32_t crc = 0xFFFFFFFF;
    for (auto byte : data) {
        crc = (crc >> 8) ^ crc32_table[(crc ^ byte) & 0xFF];
    }
    return crc ^ 0xFFFFFFFF;
}
```

**Follow-up probes:**
- Difference between `constexpr` and `consteval` (C++20)? (`consteval` forces compile-time evaluation  "immediate function")
- Can you `static_assert` on a `constexpr` function result? (Yes, if all inputs are constant expressions)
- What C++14 relaxation enabled this? (Loops and local variables in `constexpr` functions)

---

## 🔬 Q2: `std::variant` vs. Union vs. Inheritance

**Question:** You're parsing a binary protocol with 5 message types. Compare three approaches for representing a parsed message:

| Approach | Pros | Cons |
|----------|------|------|
| C-style `union` + tag | Minimal overhead, C-compatible | No constructors/destructors, type-unsafe |
| `std::variant<Msg1,...,Msg5>` | Type-safe, stack-allocated, visitor pattern | Slightly larger (index byte), no inheritance |
| Base class + virtual | Extensible, familiar OOP | Heap allocation, vtable pointer per object |

**When do you choose each in embedded?**

- **Union + tag:** When interoperating with C code or hardware registers, or when messages are POD types.
- **`std::variant`:** When messages have constructors/destructors, need type safety, and you want stack allocation. Ideal for protocol parsers.
- **Inheritance:** When the set of message types is truly open-ended (plugin architectures), or when you need runtime polymorphism across compilation units.

**Follow-up:** Implement a visitor for the variant approach:

```cpp
using Message = std::variant<Heartbeat, Telemetry, Command, Ack, Error>;

struct MessageHandler {
    void operator()(const Heartbeat& h) { update_watchdog(h.seq); }
    void operator()(const Telemetry& t) { store_sample(t); }
    void operator()(const Command& c) { execute(c); }
    void operator()(const Ack& a) { confirm(a.cmd_id); }
    void operator()(const Error& e) { log_error(e.code); }
};

void process(const Message& msg) {
    std::visit(MessageHandler{}, msg);
}
```

---

## 🔬 Q3: Structured Bindings and `std::tuple` in Driver APIs

**Question:** Redesign this legacy driver API using modern C++ (C++17+):

```cpp
// Legacy C-style
int read_sensor(float* temperature, float* humidity, uint32_t* timestamp);
// Returns 0 on success, error code on failure
```

**Expected modern approach:**

```cpp
struct SensorReading {
    float temperature;
    float humidity;
    std::chrono::steady_clock::time_point timestamp;
};

// Option 1: std::expected (C++23) or std::optional (C++17)
std::optional<SensorReading> read_sensor() {
    if (!sensor_ready()) return std::nullopt;
    return SensorReading{
        .temperature = read_temp_register(),
        .humidity = read_humidity_register(),
        .timestamp = std::chrono::steady_clock::now()
    };
}

// Usage with structured bindings
if (auto reading = read_sensor()) {
    auto [temp, humid, ts] = *reading;
    log(temp, humid, ts);
}
```

**Discussion points:**
- Designated initializers (C++20) for readability
- `std::expected<T, E>` (C++23) vs. `std::optional<T>` vs. error codes
- Why `std::optional` has zero overhead vs. pointer-based nullable (no heap, no indirection)

---

## 🔬 Q4: `if constexpr` for Platform Abstraction

**Question:** Write a compile-time platform abstraction layer that selects GPIO implementation based on target MCU family:

```cpp
enum class Platform { STM32, NRF52, ESP32, Linux };

template <Platform P>
class Gpio {
public:
    static void set_high(uint8_t pin) {
        if constexpr (P == Platform::STM32) {
            // Direct register write
            GPIOA->BSRR = (1U << pin);
        } else if constexpr (P == Platform::NRF52) {
            NRF_P0->OUTSET = (1U << pin);
        } else if constexpr (P == Platform::Linux) {
            // sysfs or libgpiod
            write_gpio_sysfs(pin, 1);
        } else {
            static_assert(always_false<P>, "Unsupported platform");
        }
    }
};

// Compile-time configuration
using HardwareGpio = Gpio<Platform::STM32>;
```

**Why `if constexpr` over preprocessor `#ifdef`?**
- Type-checked even in non-taken branches (syntax errors caught)
- No macro pollution
- Works with templates and type traits
- Debugger-friendly (no preprocessing step)

---

## 🔬 Q5: Concepts (C++20) for Embedded Interfaces

**Question:** Define a Concept that constrains a type to be a valid "Sensor Driver":

```cpp
template <typename T>
concept SensorDriver = requires(T driver, uint8_t* buf, std::size_t len) {
    { driver.init() } -> std::same_as<bool>;
    { driver.read(buf, len) } -> std::convertible_to<int>;
    { driver.reset() } noexcept;
    { T::sample_rate_hz } -> std::convertible_to<uint32_t>;
    requires sizeof(T) <= 64;  // Must fit in cache line
};

template <SensorDriver T>
class SensorManager {
public:
    bool start(T& driver) {
        if (!driver.init()) return false;
        // ...
        return true;
    }
};
```

**Why Concepts over SFINAE?**
- Readable error messages ("type X does not satisfy SensorDriver" vs. pages of template errors)
- Self-documenting interface contracts
- Subsumption rules for overload resolution
- Works with `auto` parameters: `void process(SensorDriver auto& s)`

---

## ⚡ Modern C++ Quick-Fire

| Question | Key Answer |
|----------|-----------|
| What is `std::span` (C++20)? | Non-owning view over contiguous memory (replaces `ptr + size` pairs). Zero overhead. |
| What is `[[nodiscard]]`? | Compiler warns if return value is ignored  use for error codes, locks |
| What is `std::string_view`? | Non-owning string reference  no allocation, works with C strings and `std::string` |
| What does `[[likely]]` / `[[unlikely]]` do? | Branch prediction hints for the compiler (C++20) |
| What is aggregate initialization? | Direct initialization of struct members without constructor: `Point{.x=1, .y=2}` (C++20 designated) |
| What is fold expression? | Variadic template expansion: `(args + ...)` expands parameter pack |
| What is CTAD? | Class Template Argument Deduction  `std::vector v{1,2,3}` deduces `vector<int>` (C++17) |
