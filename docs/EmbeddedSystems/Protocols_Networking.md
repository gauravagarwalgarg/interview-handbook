# Communication Protocols & Networking

**Focus: CAN, UART, SPI, I2C, REST/HTTP in embedded, MQTT, Wi-Fi/BLE stack awareness**

---

## ⚡ Quick-Fire (Protocol Gauge)

| Question | Expected Answer |
|----------|----------------|
| SPI vs I2C  when to use which? | SPI: faster (MHz), full-duplex, more wires (4+CS). I2C: slower (kHz), half-duplex, 2 wires, multi-device addressing. |
| What is a CAN bus arbitration? | Bit-dominant (0 wins). Lower message ID = higher priority. Non-destructive  winner continues transmitting. |
| What is CAN FD vs classic CAN? | CAN FD: up to 64-byte payload (vs 8), higher data-phase bitrate (up to 8 Mbps). Same arbitration. |
| UART vs RS-232 vs RS-485? | UART: logic-level protocol. RS-232: voltage levels (±12V), point-to-point. RS-485: differential, multi-drop, long distance. |
| What is I2C clock stretching? | Slave holds SCL low to pause master  "I'm not ready yet." Master must respect this. |
| What is DMA and how does it help serial comms? | DMA transfers data between peripheral and memory without CPU. Frees CPU for computation during large transfers. |
| REST vs gRPC for embedded? | REST: HTTP/JSON, simple, widely supported. gRPC: protobuf, efficient binary, streaming, code-gen. gRPC better for constrained bandwidth. |
| MQTT QoS levels? | QoS 0: at-most-once (fire-forget). QoS 1: at-least-once (ACK). QoS 2: exactly-once (4-step handshake). |
| What is mDNS/DNS-SD? | Zero-config service discovery on local network. Device announces itself without central DNS. Used by Avahi/Bonjour. |
| BLE vs Classic Bluetooth? | BLE: low power, small packets, connectionless (advertising). Classic: higher throughput, continuous connection, audio. |

---

## 🔬 Q1: CAN Bus Application Layer Design

**Scenario:** You're building a CAN-based distributed control system for an aircraft. 5 ECUs communicate over a 500 kbps CAN bus. Design the message architecture.

**Expected answer:**

```cpp
// CAN message ID scheme (11-bit standard)
// Bits 10-8: Priority (0=highest, 7=lowest)
// Bits 7-4:  Source ECU ID (0-15)
// Bits 3-0:  Message type (0-15)

enum class Priority : uint8_t { Critical = 0, High = 1, Normal = 3, Low = 5 };
enum class EcuId : uint8_t { FlightCtrl = 0, EngineCtrl = 1, NavSystem = 2, Display = 3, Logger = 4 };
enum class MsgType : uint8_t { Heartbeat = 0, SensorData = 1, Command = 2, Status = 3, Error = 4 };

constexpr uint16_t make_can_id(Priority prio, EcuId src, MsgType type) {
    return (static_cast<uint16_t>(prio) << 8)
         | (static_cast<uint16_t>(src) << 4)
         | static_cast<uint16_t>(type);
}

// Example messages
// FlightCtrl heartbeat (highest priority): ID = 0x000
// Engine sensor data: ID = 0x111
// Display command (low priority): ID = 0x532

struct CanFrame {
    uint16_t id;
    uint8_t dlc;  // Data length code (0-8 for classic CAN)
    uint8_t data[8];
    uint32_t timestamp_us;
};

// Signal packing (e.g., engine RPM in 2 bytes, big-endian)
struct EngineStatusMsg {
    static constexpr uint16_t CAN_ID = make_can_id(Priority::High, EcuId::EngineCtrl, MsgType::SensorData);
    
    uint16_t rpm;          // Bytes 0-1, big-endian, scale 0.25 RPM/bit
    int16_t  egt_celsius;  // Bytes 2-3, big-endian, scale 0.1 °C/bit
    uint16_t fuel_flow;    // Bytes 4-5, scale 0.01 L/hr/bit
    uint8_t  status_flags; // Byte 6
    uint8_t  checksum;     // Byte 7 (XOR of bytes 0-6)
    
    void serialize(CanFrame& frame) const {
        frame.id = CAN_ID;
        frame.dlc = 8;
        frame.data[0] = rpm >> 8;
        frame.data[1] = rpm & 0xFF;
        frame.data[2] = static_cast<uint16_t>(egt_celsius) >> 8;
        frame.data[3] = static_cast<uint16_t>(egt_celsius) & 0xFF;
        frame.data[4] = fuel_flow >> 8;
        frame.data[5] = fuel_flow & 0xFF;
        frame.data[6] = status_flags;
        frame.data[7] = xor_checksum(frame.data, 7);
    }
};
```

**Follow-up probes:**
- How do you handle bus-off recovery? (Auto-recovery timer, or manual reset after fault cleared)
- What is SocketCAN on Linux? (Kernel CAN framework  `AF_CAN` socket family, `can0` interface, `candump`/`cansend` tools)
- How do you test without real hardware? (Virtual CAN: `vcan0`, `ip link add dev vcan0 type vcan`)

---

## 🔬 Q2: REST API Server on Embedded Linux

**Scenario:** Your embedded device needs a local REST API for:
- Configuration management (read/write device settings)
- Live telemetry streaming (WebSocket)
- Firmware update trigger
- Device info / health status

Design the C++ HTTP server. What library do you use?

**Expected answer:**

```cpp
// Using cpp-httplib (lightweight, header-only) or Crow/Pistache for embedded
#include <httplib.h>
#include <nlohmann/json.hpp>

using json = nlohmann::json;

class DeviceRestApi {
public:
    DeviceRestApi(IConfigStore& config, ITelemetry& telemetry, uint16_t port = 8080)
        : config_(config), telemetry_(telemetry), port_(port) {}

    void start() {
        // GET /api/v1/info
        server_.Get("/api/v1/info", [this](const auto& req, auto& res) {
            json info = {
                {"device_id", get_device_id()},
                {"firmware_version", APP_VERSION},
                {"uptime_seconds", get_uptime()},
                {"cpu_temp_c", read_cpu_temp()},
            };
            res.set_content(info.dump(), "application/json");
        });

        // GET /api/v1/config
        server_.Get("/api/v1/config", [this](const auto& req, auto& res) {
            res.set_content(config_.to_json().dump(), "application/json");
        });

        // PUT /api/v1/config
        server_.Put("/api/v1/config", [this](const auto& req, auto& res) {
            try {
                auto new_config = json::parse(req.body);
                auto result = config_.apply(new_config);
                if (result) {
                    res.status = 200;
                    res.set_content(R"({"status":"applied"})", "application/json");
                } else {
                    res.status = 400;
                    res.set_content(R"({"error":"validation failed"})", "application/json");
                }
            } catch (const json::exception& e) {
                res.status = 400;
                res.set_content(json{{"error", e.what()}}.dump(), "application/json");
            }
        });

        // POST /api/v1/update
        server_.Post("/api/v1/update", [this](const auto& req, auto& res) {
            // Accept multipart firmware upload
            if (req.has_file("firmware")) {
                auto& file = req.get_file_value("firmware");
                auto result = update_manager_.stage(file.content);
                res.set_content(json{{"status", "staged"}, {"size", file.content.size()}}.dump(),
                                "application/json");
            } else {
                res.status = 400;
            }
        });

        // GET /api/v1/telemetry (SSE - Server-Sent Events)
        server_.Get("/api/v1/telemetry/stream", [this](const auto& req, auto& res) {
            res.set_header("Content-Type", "text/event-stream");
            res.set_header("Cache-Control", "no-cache");
            res.set_chunked_content_provider("text/event-stream",
                [this](size_t offset, httplib::DataSink& sink) {
                    auto data = telemetry_.get_latest();
                    std::string event = "data: " + data.to_json().dump() + "\n\n";
                    sink.write(event.c_str(), event.size());
                    std::this_thread::sleep_for(std::chrono::milliseconds(100));
                    return true;
                });
        });

        server_.listen("0.0.0.0", port_);
    }

private:
    httplib::Server server_;
    IConfigStore& config_;
    ITelemetry& telemetry_;
    uint16_t port_;
};
```

**Follow-up probes:**
- Why not use Node.js/Python for the REST server? (Single binary deployment, no runtime deps, lower memory footprint, integrates with existing C++ codebase)
- How do you secure it? (TLS via `httplib::SSLServer`, token auth, bind to localhost if only HMI access)
- What about gRPC instead? (Better for machine-to-machine, binary efficiency, but heavier runtime. REST better for browser/HMI access.)

---

## 🔬 Q3: IPC Patterns for Multi-Process Embedded Application

**Scenario:** Your embedded platform has 4 processes:
1. `sensor-daemon` (C++, RT priority  reads hardware)
2. `control-engine` (C++, normal priority  business logic)
3. `rest-api` (C++, serves web interface)
4. `ota-agent` (Python, periodic update checks)

Design the IPC between them.

**Expected answer:**

| Source → Dest | Mechanism | Why |
|---|---|---|
| sensor → control | Shared memory + eventfd | Lowest latency, zero-copy for high-rate data |
| control → rest-api | Unix domain socket (stream) | Request/response pattern, crash-isolated |
| rest-api → control | Unix domain socket | Commands from web UI to control engine |
| ota-agent → control | D-Bus (or Unix socket) | Low-rate commands, Python-friendly |
| Any → syslog | `sd_journal_send()` | Centralized structured logging |

```cpp
// Shared memory layout for sensor → control
struct SensorSharedData {
    std::atomic<uint32_t> sequence;  // Seqlock: odd = writing, even = stable
    float imu_accel[3];
    float imu_gyro[3];
    float temperature;
    float pressure;
    uint64_t timestamp_ns;
};

// Writer (sensor-daemon)
void publish(SensorSharedData* shm, const SensorReading& r) {
    shm->sequence.fetch_add(1, std::memory_order_release);  // Start write (odd)
    shm->imu_accel[0] = r.ax; /* ... */
    shm->timestamp_ns = r.ts;
    shm->sequence.fetch_add(1, std::memory_order_release);  // End write (even)
}

// Reader (control-engine)
bool read(const SensorSharedData* shm, SensorReading& out) {
    uint32_t seq1 = shm->sequence.load(std::memory_order_acquire);
    if (seq1 & 1) return false;  // Write in progress
    
    out.ax = shm->imu_accel[0]; /* ... */
    out.ts = shm->timestamp_ns;
    
    uint32_t seq2 = shm->sequence.load(std::memory_order_acquire);
    return seq1 == seq2;  // Consistent read
}
```

---

## 🔬 Q4: Wi-Fi / BLE Integration on Embedded Linux

**Question:** Your device has a Wi-Fi + BLE combo chip (e.g., Qualcomm QCA9377 or Cypress CYW43455). How do you:
1. Bring up Wi-Fi with WPA3 enterprise in Yocto?
2. Implement BLE GATT server for device configuration?
3. Handle coexistence (Wi-Fi + BLE on shared antenna)?

**Expected answers:**

**1. Wi-Fi in Yocto:**
```bitbake
# Image recipe
IMAGE_INSTALL:append = " \
    linux-firmware-qca9377 \
    wpa-supplicant \
    networkmanager \
    networkmanager-nmcli \
"

DISTRO_FEATURES:append = " wifi"

# WPA supplicant config for WPA3-Enterprise
# /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
# network={
#     ssid="CorpNetwork"
#     key_mgmt=SAE  # WPA3
#     ieee80211w=2  # MFP required
#     eap=TLS
#     identity="device@company.com"
#     ca_cert="/etc/certs/ca.pem"
#     client_cert="/etc/certs/device.pem"
#     private_key="/etc/certs/device.key"
# }
```

**2. BLE GATT server (using BlueZ D-Bus API):**
```python
# Python BLE GATT server for device configuration
import dbus
import dbus.service
from gi.repository import GLib

DEVICE_CONFIG_SERVICE_UUID = '12345678-1234-5678-1234-56789abcdef0'
WIFI_SSID_CHAR_UUID        = '12345678-1234-5678-1234-56789abcdef1'

class WifiSsidCharacteristic(dbus.service.Object):
    """BLE characteristic: write SSID to configure Wi-Fi."""
    
    def ReadValue(self, options):
        return dbus.Array(self.current_ssid.encode(), signature='y')
    
    def WriteValue(self, value, options):
        ssid = bytes(value).decode()
        self.configure_wifi(ssid)
```

**3. Coexistence:** Handled by firmware-level coex tables (TDMA arbitration between Wi-Fi and BLE). Linux side: ensure `btcoex` module is loaded, use `iw reg set` for correct regulatory domain.

---

## ⚡ Protocol Comparison Matrix

| Protocol | Speed | Topology | Distance | Use Case |
|----------|-------|----------|----------|----------|
| SPI | 1-100 MHz | Star (1 master, N slaves) | PCB-level (cm) | Fast sensors, FPGA, Flash |
| I2C | 100-400 kHz (3.4 MHz HS) | Multi-master bus | Short (m) | Config EEPROMs, slow sensors |
| UART | 9600-4 Mbps | Point-to-point | Short (m) with RS-232 | GPS, debug console, modems |
| CAN | 125 kbps-1 Mbps | Multi-master bus | 40m@1Mbps, 1km@125kbps | Automotive, industrial |
| CAN FD | Up to 8 Mbps data phase | Multi-master bus | Shorter than classic CAN | Modern automotive |
| Ethernet | 10/100/1000 Mbps | Switched/P2P | 100m (copper) | High-bandwidth, IP-based |
| MQTT | N/A (over TCP) | Pub/sub via broker | WAN | IoT cloud telemetry |
| BLE | 1-2 Mbps | Star (central/peripheral) | ~50m | Config, beacons, low-power |
| Wi-Fi | Up to Gbps | Star (AP/STA) | ~100m | High-bandwidth local comms |
