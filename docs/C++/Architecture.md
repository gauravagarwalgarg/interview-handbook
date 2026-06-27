# C++ System Architecture  Senior/Staff Level (9+ Years)

**Focus: Large-scale embedded platform design, API boundaries, layered architecture, build-time vs runtime decisions, testability at scale**

---

## 🏗️ Q1: Layered Platform Architecture (20 min)

**Scenario:** You are the platform architect for a Zynq-based industrial controller.
The system has:
- **PL (FPGA fabric):** Motor control loops at 100 kHz, custom DSP pipelines
- **PS (ARM Cortex-A53):** Linux with application layer
- **Communication:** EtherCAT master, MQTT to cloud, local HMI web interface

Design the C++ software architecture for the PS side.

**Expected layered design:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Application Layer                              │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ Motion Planner│  │ Recipe Engine │  │ Diagnostics Manager   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬────────────┘  │
├─────────┼──────────────────┼─────────────────────┼───────────────┤
│         │     Service Layer (Domain Logic)        │               │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────────▼────────────┐  │
│  │MotionService │  │ProcessService│  │  HealthService         │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬────────────┘  │
├─────────┼──────────────────┼─────────────────────┼───────────────┤
│         │     Platform Abstraction Layer (PAL)    │               │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────────▼────────────┐  │
│  │ FpgaComm     │  │ EtherCATMgr  │  │  MqttClient           │  │
│  │ (AXI/UIO)    │  │              │  │                        │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬────────────┘  │
├─────────┼──────────────────┼─────────────────────┼───────────────┤
│         │         HAL / OS Abstraction            │               │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────────▼────────────┐  │
│  │ UIO Driver   │  │ Socket/Net   │  │  FileSystem            │  │
│  │ /dev/uio0    │  │              │  │                        │  │
│  └──────────────┘  └──────────────┘  └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Key architectural decisions to evaluate:**

| Decision | Senior Answer |
|----------|--------------|
| Why layers? | Testability (mock lower layers), portability (swap HAL for simulation), team boundaries |
| API boundaries | Each layer exposes pure interfaces (`IMotionService`), implementation is internal |
| Dependency direction | Always downward. Upper layers never `#include` lower layer internals. |
| FPGA communication | Memory-mapped registers via UIO. Wrap in typed C++ API with `volatile` semantics. |
| Error propagation | `std::expected` or error codes across layer boundaries. No exceptions crossing API. |
| Threading model | One real-time thread per critical path. Services communicate via lock-free queues. |
| Configuration | Compile-time for HW-dependent (constexpr), runtime for operational params (JSON/YAML) |

**Follow-up probes:**
- How do you enforce layer boundaries at build time? (Separate CMake libraries per layer, no upward `target_link_libraries`)
- How do you version the FPGA ↔ PS interface? (Register map header generated from FPGA build, version register at offset 0x00)
- What if the FPGA register map changes? (Adapter pattern at PAL layer absorbs changes, application layer unchanged)

---

## 🏗️ Q2: API Design for Multi-Team Development (15 min)

**Scenario:** Three teams work on this platform:
- **FPGA Team:** Delivers bitstream + register map specification
- **Platform Team:** Owns HAL, PAL, services (your team)
- **Application Team:** Builds customer-facing features

Design the C++ API contract between Platform and Application teams.

**Expected answer:**

```cpp
// Public API header  this is the contract
// platform/include/platform/motion_service.h

#pragma once
#include <cstdint>
#include <span>
#include <expected>
#include <chrono>

namespace platform {

enum class MotionError : uint8_t {
    Ok = 0, NotHomed, LimitReached, FpgaComm, Timeout, InvalidParam
};

struct MotionProfile {
    float velocity_mm_s;
    float acceleration_mm_s2;
    float deceleration_mm_s2;
    float jerk_mm_s3;  // S-curve
};

struct Position {
    double x_mm;
    double y_mm;
    double z_mm;
};
```


```cpp
// Abstract interface  Application team programs against this
class IMotionService {
public:
    virtual ~IMotionService() = default;

    [[nodiscard]] virtual std::expected<void, MotionError>
        move_absolute(Position target, MotionProfile profile) = 0;

    [[nodiscard]] virtual std::expected<void, MotionError>
        move_relative(Position delta, MotionProfile profile) = 0;

    [[nodiscard]] virtual std::expected<Position, MotionError>
        get_position() const = 0;

    [[nodiscard]] virtual std::expected<void, MotionError>
        home(std::chrono::seconds timeout = std::chrono::seconds{30}) = 0;

    virtual void halt() noexcept = 0;  // Emergency stop  never fails

    // Event subscription
    using MotionCompleteCallback = void(*)(Position final_pos, void* ctx);
    virtual void on_complete(MotionCompleteCallback cb, void* ctx) = 0;
};

// Factory  Application team doesn't know concrete type
[[nodiscard]] std::unique_ptr<IMotionService>
    create_motion_service(const PlatformConfig& config);

}  // namespace platform
```

**Evaluation criteria for senior candidate:**

| Aspect | What to look for |
|--------|-----------------|
| Stability | Interface is `std::expected`-based  adding error codes doesn't break ABI |
| ABI safety | Uses abstract interface + factory  implementation can change without recompiling app |
| Thread safety contract | Document: "All methods are thread-safe. `halt()` can be called from any thread." |
| Testability | Application team can mock `IMotionService` for unit tests without hardware |
| Versioning | `PlatformConfig` carries API version  graceful degradation for older apps |

**Follow-up probes:**
- How do you handle ABI stability across shared library versions? (Pimpl idiom, COM-style interface, or `dlopen` with C API)
- What about `std::expected` across shared library boundaries? (Risky  prefer C-compatible error codes at .so boundary, wrap internally)
- How do you document thread-safety guarantees? (Annotations: `[[thread_safe]]`, TSan CI, or contract comments)

---

## 🏗️ Q3: Dependency Injection & Testability (15 min)

**Scenario:** Your sensor fusion module depends on:
- IMU driver (hardware)
- GPS driver (hardware)
- Time source (system clock)
- Logger (file/network)
- Configuration store (flash/file)

How do you structure this so it's testable on a developer laptop without any hardware?

**Expected answer  Constructor injection with interfaces:**

```cpp
// Interfaces for all external dependencies
class IImuDriver {
public:
    virtual ~IImuDriver() = default;
    virtual std::expected<ImuSample, DriverError> read() = 0;
    virtual bool init() = 0;
};

class IGpsDriver {
public:
    virtual ~IGpsDriver() = default;
    virtual std::expected<GpsPosition, DriverError> get_fix() = 0;
};

class IClock {
public:
    virtual ~IClock() = default;
    virtual std::chrono::steady_clock::time_point now() const = 0;
};

class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(Severity s, std::string_view msg) = 0;
};
```


```cpp
// Fusion engine with all dependencies injected
class SensorFusionEngine {
public:
    struct Dependencies {
        IImuDriver& imu;
        IGpsDriver& gps;
        IClock& clock;
        ILogger& logger;
    };

    explicit SensorFusionEngine(Dependencies deps, FusionConfig config)
        : deps_(deps), config_(config) {}

    void run_cycle() {
        auto imu_result = deps_.imu.read();
        if (!imu_result) {
            deps_.logger.log(Severity::Error, "IMU read failed");
            return;
        }

        auto now = deps_.clock.now();
        auto dt = std::chrono::duration<float>(now - last_update_).count();
        last_update_ = now;

        kalman_.predict(imu_result->accel, imu_result->gyro, dt);

        // GPS update at lower rate
        if (auto gps_fix = deps_.gps.get_fix()) {
            kalman_.update_gps(*gps_fix);
        }
    }

    NavigationState get_state() const { return kalman_.state(); }

private:
    Dependencies deps_;
    FusionConfig config_;
    KalmanFilter kalman_;
    std::chrono::steady_clock::time_point last_update_{};
};

// Production wiring
RealImuDriver real_imu("/dev/spidev0.0");
RealGpsDriver real_gps("/dev/ttyS1");
SystemClock sys_clock;
SyslogLogger logger;

SensorFusionEngine engine(
    {real_imu, real_gps, sys_clock, logger},
    load_config("/etc/fusion.conf")
);

// Test wiring  runs on laptop, no hardware needed
MockImu mock_imu;
mock_imu.set_samples(load_recorded_data("test_flight.bin"));
MockGps mock_gps;
FakeClock fake_clock;  // Controllable time
NullLogger null_log;

SensorFusionEngine test_engine(
    {mock_imu, mock_gps, fake_clock, null_log},
    default_config()
);
```

**Why this matters for 9+ year candidates:**
- They should naturally reach for DI without prompting
- They should discuss trade-offs: virtual dispatch cost vs. testability (acceptable on A53, not on M0)
- They should mention compile-time DI alternatives: templates + concepts for zero-overhead
- They should know the "Composition Root" pattern (wire once at main(), not scattered)

---

## 🏗️ Q4: Plugin Architecture for Extensible Systems (15 min)

**Scenario:** Your industrial controller platform needs to support
customer-specific plugins that are:
- Loaded at runtime from `/opt/plugins/*.so`
- Isolated (plugin crash doesn't kill platform)
- Versioned (platform API v2 can load v1 and v2 plugins)
- Sandboxed (plugins can't access arbitrary hardware)

Design the plugin framework.

**Expected answer:**

```cpp
// Plugin ABI  C interface for .so boundary (no C++ ABI issues)
extern "C" {

struct PluginInfo {
    uint32_t api_version;       // Plugin was compiled against this API version
    const char* name;
    const char* version;
    const char* author;
};

// Plugin lifecycle functions
typedef PluginInfo (*plugin_get_info_fn)();
typedef int (*plugin_init_fn)(void* platform_context);
typedef void (*plugin_shutdown_fn)();
typedef int (*plugin_execute_fn)(const char* command, const char* params, char* response, size_t max_response);

}  // extern "C"

// Platform-side plugin loader
class PluginManager {
public:
    struct LoadedPlugin {
        void* handle = nullptr;  // dlopen handle
        PluginInfo info{};
        plugin_init_fn init = nullptr;
        plugin_shutdown_fn shutdown = nullptr;
        plugin_execute_fn execute = nullptr;
        bool active = false;
    };

    bool load_all(const std::filesystem::path& plugin_dir) {
        for (const auto& entry : std::filesystem::directory_iterator(plugin_dir)) {
            if (entry.path().extension() == ".so") {
                load_plugin(entry.path());
            }
        }
        return true;
    }
```


```cpp
private:
    bool load_plugin(const std::filesystem::path& path) {
        void* handle = dlopen(path.c_str(), RTLD_LAZY | RTLD_LOCAL);
        if (!handle) {
            log_error("dlopen failed: {}", dlerror());
            return false;
        }

        auto get_info = (plugin_get_info_fn)dlsym(handle, "plugin_get_info");
        if (!get_info) { dlclose(handle); return false; }

        PluginInfo info = get_info();
        if (info.api_version > CURRENT_API_VERSION) {
            log_error("Plugin {} requires API v{}, we are v{}",
                      info.name, info.api_version, CURRENT_API_VERSION);
            dlclose(handle);
            return false;
        }

        LoadedPlugin plugin{
            .handle = handle,
            .info = info,
            .init = (plugin_init_fn)dlsym(handle, "plugin_init"),
            .shutdown = (plugin_shutdown_fn)dlsym(handle, "plugin_shutdown"),
            .execute = (plugin_execute_fn)dlsym(handle, "plugin_execute"),
        };

        if (plugin.init && plugin.init(&platform_ctx_) == 0) {
            plugin.active = true;
            plugins_.push_back(plugin);
            return true;
        }
        dlclose(handle);
        return false;
    }

    static constexpr uint32_t CURRENT_API_VERSION = 2;
    std::vector<LoadedPlugin> plugins_;
    PlatformContext platform_ctx_;
};
```

**Key discussion points for senior candidates:**
- Why C ABI at .so boundary? (C++ name mangling, vtable layout, std::string ABI differ between compilers/versions)
- How to isolate plugin crashes? (Separate process per plugin, communicate via IPC. Or: signal handler + `longjmp` for less isolation)
- How to sandbox? (seccomp-bpf, separate user/group, capability dropping, or run in container/namespace)
- Versioning strategy? (Additive-only API changes. Old plugins see subset of new context. Never remove fields.)

---

## 🏗️ Q5: Compile-Time Configuration vs Runtime Configuration (10 min)

**Scenario:** Your platform supports 5 hardware variants (different Zynq models,
different peripheral sets). How do you handle variant-specific code?

**Expected answer  Layered approach:**

```cpp
// 1. COMPILE-TIME: Hardware-dependent constants (known at build time)
// Generated from Yocto MACHINE variable or CMake option
namespace hw_config {
    // Set by CMake -DHW_VARIANT=ZYNQ_7020
    #if HW_VARIANT == ZYNQ_7020
        constexpr uint32_t fpga_base_addr = 0x4000'0000;
        constexpr uint32_t axi_dma_channels = 4;
        constexpr bool has_ethernet_phy_b = false;
    #elif HW_VARIANT == ZYNQMP_ZU9
        constexpr uint32_t fpga_base_addr = 0xA000'0000;
        constexpr uint32_t axi_dma_channels = 8;
        constexpr bool has_ethernet_phy_b = true;
    #endif
}

// 2. COMPILE-TIME: Feature selection via if constexpr
template <typename Platform>
class NetworkManager {
public:
    void init() {
        init_phy_a();
        if constexpr (Platform::has_dual_phy) {
            init_phy_b();  // Only compiled for ZU9
        }
    }
};

// 3. RUNTIME: Operational parameters (customer-configurable)
struct RuntimeConfig {
    float control_loop_hz = 1000.0f;   // Tunable
    uint8_t motor_count = 4;            // Detected at boot
    std::string mqtt_broker = "mqtt.factory.local";
    bool enable_cloud_telemetry = true;
};
// Loaded from /etc/platform.json or provisioned via cloud
```

**Decision matrix:**

| Config Type | When | Mechanism | Example |
|-------------|------|-----------|---------|
| Hardware variant | Build time | CMake + `#if` / `if constexpr` | Register addresses, DMA channels |
| Feature flags | Build time | Yocto `DISTRO_FEATURES` → CMake | Enable/disable subsystems |
| Operational tuning | Runtime | JSON/YAML config file | PID gains, network addresses |
| Customer secrets | Provisioning | Secure element / TPM | TLS certs, API keys |

---

## 🏗️ Q6: Error Handling Strategy at Scale (10 min)

**Question:** Define the error handling strategy for a platform with 50+ source files,
3 teams, and a mix of real-time and non-real-time paths.

**Expected senior-level answer:**

```cpp
// 1. Define error domains per layer
namespace platform::error {
    enum class Fpga : uint8_t { Ok, Timeout, CrcMismatch, NotReady, Overrun };
    enum class Network : uint8_t { Ok, Timeout, Refused, DnsFailure, TlsError };
    enum class Motion : uint8_t { Ok, NotHomed, LimitHit, FollowingError, EStop };
}

// 2. Use std::expected at API boundaries
using FpgaResult = std::expected<FpgaResponse, error::Fpga>;
using MotionResult = std::expected<void, error::Motion>;

// 3. Error context for debugging (non-real-time paths only)
class ErrorContext {
public:
    static ErrorContext& last() {
        thread_local ErrorContext instance;
        return instance;
    }
    void set(std::string_view file, int line, std::string_view detail) {
        file_ = file; line_ = line; detail_ = detail;
    }
    // ... accessors
private:
    std::string_view file_;
    int line_ = 0;
    std::string_view detail_;
};

#define PLATFORM_ERROR(err, detail) \
    (ErrorContext::last().set(__FILE__, __LINE__, detail), \
     std::unexpected(err))

// 4. Fatal error policy  platform-level decision
class FatalHandler {
public:
    [[noreturn]] static void panic(std::string_view reason) {
        // Log to persistent storage
        // Set boot reason flag for post-mortem
        // Trigger watchdog reset (or enter safe mode)
        log_to_flash(reason);
        set_boot_reason(BootReason::Panic);
        while(true) { __WFI(); }  // Wait for watchdog
    }
};
```

**Key questions for senior:**
- Where do you draw the line between recoverable and fatal? (Lost FPGA comm = fatal. Lost MQTT = recoverable with retry.)
- How do you avoid error code sprawl? (Per-domain enums, not one giant enum. Map to string only at boundaries.)
- How do you test error paths? (Fault injection framework  mock drivers return errors on demand)

---

## ⚡ Architecture Quick-Fire (Senior Gauge)

| Question | Expected Senior Answer |
|----------|----------------------|
| What is the Pimpl idiom and when do you use it? | Pointer-to-implementation. Hides private members from header. Reduces recompilation. Use at public API / .so boundaries. |
| What is type erasure? | Hiding concrete type behind value-semantic interface (like `std::function`, `std::any`). Enables polymorphism without inheritance. |
| How do you enforce architectural layers at build time? | Separate CMake targets per layer. Private includes. No upward `target_link_libraries`. CI check for forbidden includes. |
| What is the Hexagonal Architecture? | Ports & Adapters. Core domain has no external deps. Adapters plug in (DB, network, HW). Core is 100% testable. |
| Monorepo or multirepo for platform + apps? | Monorepo with Bazel/CMake targets for build isolation. Single atomic commits. Multirepo only if teams are truly decoupled. |
| How do you handle backward compatibility in a platform library? | Semantic versioning. Add, never remove. Use opaque handles. Version check at load. Deprecation cycle. |
| What is the cost of virtual dispatch in a hot loop? | ~3-5ns per call (indirect branch + possible I-cache miss). Unacceptable at 100 kHz. Use CRTP or templates for hot paths. |
