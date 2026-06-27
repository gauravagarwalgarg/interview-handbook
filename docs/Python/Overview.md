# Python Scripting Interview Questions

**Focus: Automation, Log Parsing, Serial Communication, Hardware Interfaces, pytest, CI/CD Integration**

---

## ⚡ Quick-Fire (Python Fluency Gauge)

| Question | Expected Answer |
|----------|----------------|
| Difference between `list` and `tuple`? | List: mutable, variable size. Tuple: immutable, fixed, hashable (can be dict key). |
| What is a generator and why use it? | Lazy iterator using `yield`. Memory-efficient for large datasets  doesn't hold all values in memory. |
| What is the GIL? | Global Interpreter Lock  only one thread executes Python bytecode at a time. Use `multiprocessing` for CPU-bound parallelism. |
| How do you handle binary data? | `struct.pack()`/`struct.unpack()` for fixed layouts, `bytes`/`bytearray` for raw buffers |
| What is a context manager? | `with` statement protocol (`__enter__`/`__exit__`). Guarantees cleanup (files, locks, connections). |
| What is `subprocess.run` vs `Popen`? | `run`: blocking, returns CompletedProcess. `Popen`: non-blocking, gives you stdin/stdout streams. |
| What is type hinting and does it enforce types? | Annotations for readability/tools. Not enforced at runtime  use `mypy` for static checking. |
| What is `pathlib` vs `os.path`? | `pathlib`: OOP, cross-platform path manipulation. Preferred in modern Python (3.4+). |

---

## 🔬 Q1: Serial Communication  Sensor Data Acquisition

**Question:** Write a Python script that:
1. Opens a serial port (`/dev/ttyUSB0`, 115200 baud)
2. Reads binary frames (header: `0xAA 0x55`, 2-byte length, payload, 1-byte CRC)
3. Validates CRC
4. Logs parsed telemetry to a CSV file
5. Handles port disconnection gracefully

**Expected answer:**

```python
import serial
import struct
import csv
import logging
from pathlib import Path
from dataclasses import dataclass
from typing import Optional

logger = logging.getLogger(__name__)

SYNC_HEADER = bytes([0xAA, 0x55])

@dataclass
class TelemetryFrame:
    timestamp_ms: int
    temperature: float
    pressure: float
    battery_mv: int

def calculate_crc(data: bytes) -> int:
    """Simple XOR CRC-8."""
    crc = 0
    for byte in data:
        crc ^= byte
    return crc

def parse_frame(payload: bytes) -> Optional[TelemetryFrame]:
    """Parse 14-byte telemetry payload."""
    if len(payload) != 14:
        return None
    ts, temp, press, batt = struct.unpack('<I f f H', payload)
    return TelemetryFrame(
        timestamp_ms=ts,
        temperature=temp,
        pressure=press,
        battery_mv=batt,
    )

def read_frames(port: str, baudrate: int = 115200):
    """Generator that yields validated telemetry frames."""
    while True:
        try:
            with serial.Serial(port, baudrate, timeout=1.0) as ser:
                logger.info(f"Connected to {port}")
                buffer = bytearray()

                while True:
                    chunk = ser.read(ser.in_waiting or 1)
                    if not chunk:
                        continue
                    buffer.extend(chunk)

                    while len(buffer) >= 4:  # Minimum: header(2) + length(2)
                        # Find sync header
                        idx = buffer.find(SYNC_HEADER)
                        if idx == -1:
                            buffer.clear()
                            break
                        if idx > 0:
                            buffer = buffer[idx:]

                        if len(buffer) < 4:
                            break

                        length = struct.unpack_from('<H', buffer, 2)[0]
                        frame_size = 2 + 2 + length + 1  # header + len + payload + crc

                        if len(buffer) < frame_size:
                            break

                        payload = bytes(buffer[4:4 + length])
                        received_crc = buffer[4 + length]
                        expected_crc = calculate_crc(payload)

                        if received_crc == expected_crc:
                            frame = parse_frame(payload)
                            if frame:
                                yield frame
                        else:
                            logger.warning(f"CRC mismatch: got {received_crc:#x}, expected {expected_crc:#x}")

                        buffer = buffer[frame_size:]

        except serial.SerialException as e:
            logger.error(f"Serial error: {e}. Reconnecting in 2s...")
            import time
            time.sleep(2)

def main():
    output = Path("telemetry.csv")
    with output.open("w", newline="") as csvfile:
        writer = csv.writer(csvfile)
        writer.writerow(["timestamp_ms", "temperature", "pressure", "battery_mv"])

        for frame in read_frames("/dev/ttyUSB0"):
            writer.writerow([frame.timestamp_ms, frame.temperature, frame.pressure, frame.battery_mv])
            csvfile.flush()
            logger.info(f"T={frame.temperature:.1f}°C P={frame.pressure:.0f}Pa Batt={frame.battery_mv}mV")

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    main()
```

**Follow-up probes:**
- How do you unit test this without real hardware? (Mock `serial.Serial` with `unittest.mock`, feed known byte sequences)
- How do you handle a frame that spans two `read()` calls? (Buffer accumulation  already shown above)
- What if the sensor sends data faster than Python can process? (Increase OS serial buffer, use separate reader thread with queue)

---

## 🔬 Q2: Log Parsing and Analysis

**Question:** You have 500 MB of kernel `dmesg` logs from a field test. Write a script that:
1. Extracts all lines matching `[ERROR]` or `[WARN]` with timestamps
2. Groups errors by subsystem (e.g., `usb`, `mmc`, `net`)
3. Generates a summary report with frequency and first/last occurrence
4. Outputs both JSON and human-readable format

**Expected answer:**

```python
import re
from collections import defaultdict
from dataclasses import dataclass, field
from pathlib import Path
import json
from typing import Iterator

@dataclass
class ErrorEntry:
    count: int = 0
    first_seen: str = ""
    last_seen: str = ""
    samples: list[str] = field(default_factory=list)

# Pattern: [    3.456789] subsystem: [ERROR] message
LOG_PATTERN = re.compile(
    r'\[\s*(?P<timestamp>[\d.]+)\]\s+'
    r'(?P<subsystem>\w+).*?'
    r'\[(?P<level>ERROR|WARN)\]\s+'
    r'(?P<message>.+)'
)

def parse_log_lines(filepath: Path) -> Iterator[dict]:
    """Stream-parse log file without loading entirely into memory."""
    with filepath.open('r', errors='replace') as f:
        for line in f:
            m = LOG_PATTERN.search(line)
            if m:
                yield m.groupdict()

def analyze_logs(filepath: Path) -> dict[str, ErrorEntry]:
    summary: dict[str, ErrorEntry] = defaultdict(ErrorEntry)

    for entry in parse_log_lines(filepath):
        key = f"{entry['subsystem']}:{entry['level']}"
        record = summary[key]
        record.count += 1
        if not record.first_seen:
            record.first_seen = entry['timestamp']
        record.last_seen = entry['timestamp']
        if len(record.samples) < 3:
            record.samples.append(entry['message'].strip())

    return dict(summary)

def generate_report(summary: dict[str, ErrorEntry], output_dir: Path):
    # JSON output
    json_data = {
        k: {"count": v.count, "first": v.first_seen, "last": v.last_seen, "samples": v.samples}
        for k, v in sorted(summary.items(), key=lambda x: x[1].count, reverse=True)
    }
    (output_dir / "error_summary.json").write_text(json.dumps(json_data, indent=2))

    # Human-readable
    lines = ["=" * 60, "ERROR/WARN SUMMARY", "=" * 60, ""]
    for key, entry in sorted(summary.items(), key=lambda x: x[1].count, reverse=True):
        lines.append(f"{key:30s} | Count: {entry.count:5d} | First: {entry.first_seen} | Last: {entry.last_seen}")
        for sample in entry.samples:
            lines.append(f"    → {sample}")
        lines.append("")

    report_text = "\n".join(lines)
    (output_dir / "error_summary.txt").write_text(report_text)
    print(report_text)

if __name__ == "__main__":
    import sys
    log_path = Path(sys.argv[1]) if len(sys.argv) > 1 else Path("dmesg.log")
    output = Path("reports")
    output.mkdir(exist_ok=True)
    summary = analyze_logs(log_path)
    generate_report(summary, output)
```

**Follow-up probes:**
- Why use a generator instead of loading the file? (500 MB file  memory efficiency)
- How would you parallelize for multiple log files? (`concurrent.futures.ProcessPoolExecutor`)
- How do you handle compressed log files (`.gz`)? (`gzip.open()` with same interface)

---

## 🔬 Q3: pytest for Hardware-in-the-Loop Testing

**Question:** Write a pytest test suite that validates a C++ sensor daemon running on target hardware. Tests should:
1. Verify the daemon starts and creates its Unix socket
2. Send a "read temperature" command and validate response format
3. Inject a fault (disconnect sensor) and verify error reporting
4. Measure response latency (must be < 50 ms)

**Expected answer:**

```python
import pytest
import socket
import struct
import time
import subprocess
from pathlib import Path

SOCKET_PATH = "/tmp/sensor-daemon.sock"
TIMEOUT_S = 2.0

@pytest.fixture(scope="module")
def daemon_process():
    """Start the sensor daemon for the test session."""
    proc = subprocess.Popen(
        ["/usr/bin/sensor-daemon", "--config", "/etc/sensor.conf"],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
    )
    # Wait for socket to appear
    deadline = time.monotonic() + TIMEOUT_S
    while time.monotonic() < deadline:
        if Path(SOCKET_PATH).exists():
            break
        time.sleep(0.05)
    else:
        proc.kill()
        pytest.fail("Daemon did not create socket within timeout")

    yield proc

    proc.terminate()
    proc.wait(timeout=5)

@pytest.fixture
def client_socket(daemon_process):
    """Connect to the daemon's Unix socket."""
    sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    sock.settimeout(TIMEOUT_S)
    sock.connect(SOCKET_PATH)
    yield sock
    sock.close()

def send_command(sock: socket.socket, cmd_id: int, payload: bytes = b"") -> bytes:
    """Send command frame and receive response."""
    frame = struct.pack('<BH', cmd_id, len(payload)) + payload
    sock.sendall(frame)
    header = sock.recv(3)
    resp_id, resp_len = struct.unpack('<BH', header)
    data = sock.recv(resp_len)
    return data

class TestDaemonStartup:
    def test_socket_exists(self, daemon_process):
        assert Path(SOCKET_PATH).exists()

    def test_daemon_pid_valid(self, daemon_process):
        assert daemon_process.poll() is None  # Still running

class TestTemperatureRead:
    CMD_READ_TEMP = 0x01

    def test_response_format(self, client_socket):
        """Response should be 4-byte float (temperature in Celsius)."""
        data = send_command(client_socket, self.CMD_READ_TEMP)
        assert len(data) == 4
        temp = struct.unpack('<f', data)[0]
        assert -40.0 <= temp <= 125.0  # Valid sensor range

    def test_response_latency(self, client_socket):
        """Response must arrive within 50 ms."""
        start = time.monotonic()
        send_command(client_socket, self.CMD_READ_TEMP)
        elapsed_ms = (time.monotonic() - start) * 1000
        assert elapsed_ms < 50.0, f"Latency too high: {elapsed_ms:.1f} ms"

    @pytest.mark.parametrize("iteration", range(100))
    def test_repeated_reads_stable(self, client_socket, iteration):
        """100 consecutive reads should not crash or timeout."""
        data = send_command(client_socket, self.CMD_READ_TEMP)
        assert len(data) == 4

class TestFaultInjection:
    CMD_INJECT_FAULT = 0xF0
    CMD_READ_STATUS = 0x02

    def test_sensor_disconnect_reported(self, client_socket):
        """After fault injection, status should report error."""
        # Inject simulated sensor disconnect
        send_command(client_socket, self.CMD_INJECT_FAULT, b'\x01')
        time.sleep(0.1)

        status_data = send_command(client_socket, self.CMD_READ_STATUS)
        status_code = struct.unpack('<I', status_data)[0]
        assert status_code != 0, "Expected error status after fault injection"
```

**Follow-up probes:**
- How do you run these tests on the target from a CI pipeline? (SSH + pytest-remote, or pytest on target via NFS rootfs)
- How do you mock hardware for local development? (Fake daemon mode with `--mock` flag, or socket-based mock server)
- What pytest plugins help? (`pytest-timeout`, `pytest-repeat`, `pytest-html` for reports)

---

## 🔬 Q4: Build System Automation Script

**Question:** Write a Python script that orchestrates a full CI pipeline:
1. Pulls latest Yocto layers (pins from `kas.yml`)
2. Triggers bitbake build
3. Runs QEMU-based smoke tests on the resulting image
4. Uploads artifacts to an artifact server
5. Posts build status to GitLab

**Expected high-level structure:**

```python
#!/usr/bin/env python3
"""CI pipeline orchestrator for Yocto-based embedded product."""

import subprocess
import sys
import json
import shutil
from pathlib import Path
from dataclasses import dataclass
from enum import Enum, auto

class BuildStatus(Enum):
    SUCCESS = auto()
    BUILD_FAILED = auto()
    TEST_FAILED = auto()
    UPLOAD_FAILED = auto()

@dataclass
class BuildResult:
    status: BuildStatus
    image_path: Path | None = None
    test_report: Path | None = None
    log: str = ""

def run_cmd(cmd: list[str], cwd: Path | None = None, timeout: int = 3600) -> subprocess.CompletedProcess:
    """Run command with timeout and logging."""
    print(f"  → {' '.join(cmd)}")
    return subprocess.run(cmd, cwd=cwd, capture_output=True, text=True, timeout=timeout, check=True)

def build_image(build_dir: Path) -> Path:
    """Run kas build and return image path."""
    run_cmd(["kas", "build", "kas.yml"], cwd=build_dir)
    # Find generated image
    deploy = build_dir / "build" / "tmp" / "deploy" / "images"
    images = list(deploy.rglob("*.wic.gz"))
    if not images:
        raise FileNotFoundError("No .wic.gz image found in deploy directory")
    return images[0]

def run_qemu_tests(image_path: Path, test_dir: Path) -> Path:
    """Boot image in QEMU and run pytest suite."""
    report_path = test_dir / "report.xml"
    run_cmd([
        "pytest", str(test_dir / "tests"),
        f"--image={image_path}",
        f"--junitxml={report_path}",
        "--timeout=300",
    ])
    return report_path

def upload_artifacts(image_path: Path, report_path: Path, server_url: str):
    """Upload build artifacts to company artifact server."""
    import requests
    for artifact in [image_path, report_path]:
        with artifact.open('rb') as f:
            resp = requests.post(f"{server_url}/upload", files={"file": f})
            resp.raise_for_status()

def notify_gitlab(status: BuildStatus, pipeline_url: str):
    """Post build status back to GitLab merge request."""
    import requests
    import os
    state_map = {BuildStatus.SUCCESS: "success", BuildStatus.BUILD_FAILED: "failed", BuildStatus.TEST_FAILED: "failed"}
    requests.post(
        f"{os.environ['CI_API_V4_URL']}/projects/{os.environ['CI_PROJECT_ID']}/statuses/{os.environ['CI_COMMIT_SHA']}",
        headers={"PRIVATE-TOKEN": os.environ['GITLAB_TOKEN']},
        json={"state": state_map.get(status, "failed"), "target_url": pipeline_url},
    )

def main() -> int:
    build_dir = Path.cwd()
    try:
        image = build_image(build_dir)
        report = run_qemu_tests(image, build_dir)
        upload_artifacts(image, report, "https://artifacts.company.com")
        notify_gitlab(BuildStatus.SUCCESS, "")
        return 0
    except subprocess.CalledProcessError as e:
        print(f"FAILED: {e.cmd}\nSTDERR: {e.stderr[-2000:]}")
        notify_gitlab(BuildStatus.BUILD_FAILED, "")
        return 1

if __name__ == "__main__":
    sys.exit(main())
```

---

## 🔬 Q5: Python ↔ C++ Integration

**Question:** You need to call a C++ shared library from Python for post-processing sensor data. Compare approaches:

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| `ctypes` | No compilation, stdlib | Manual type marshaling, fragile | Simple C-compatible APIs |
| `cffi` | Clean API, ABI + API modes | Extra dependency | C-compatible APIs with complexity |
| `pybind11` | Full C++ support, Pythonic | Requires compilation | Complex C++ classes, templates |
| `subprocess` | No binding needed | Process overhead, serialization cost | CLI tools, one-shot analysis |

**Example with ctypes:**

```python
import ctypes
from ctypes import c_float, c_uint32, Structure, POINTER

class SensorData(Structure):
    _fields_ = [
        ("timestamp_ms", c_uint32),
        ("temperature", c_float),
        ("pressure", c_float),
    ]

lib = ctypes.CDLL("./libsensor_processing.so")
lib.process_samples.argtypes = [POINTER(SensorData), c_uint32]
lib.process_samples.restype = c_float

# Create array of samples
samples = (SensorData * 100)()
# ... fill samples ...
result = lib.process_samples(samples, 100)
```

---

## ⚡ Additional Quick Checks

| Question | Key Answer |
|----------|-----------|
| How do you make a Python script executable on Linux? | `chmod +x script.py` + shebang `#!/usr/bin/env python3` |
| What is `venv` and why use it? | Isolated Python environment  prevents system package conflicts |
| How do you profile a slow script? | `cProfile`, `line_profiler`, or `py-spy` for sampling profiler |
| What is `asyncio` and when to use it? | Cooperative concurrency for I/O-bound tasks (network, serial). Not for CPU-bound. |
| How do you parse CLI args properly? | `argparse` (stdlib) or `click` (third-party, decorator-based) |
| What is `dataclass` vs `NamedTuple`? | Dataclass: mutable by default, richer features. NamedTuple: immutable, lighter, tuple-compatible. |
