# Cross-Functional Scenario Questions

**Focus: End-to-end problem solving where C++, Embedded Linux, Yocto, and Python intersect**

---

## 🏗️ Scenario 1: Field Update System (20 min)

**Context:** You are building an OTA (Over-The-Air) update system for a fleet of IoT gateways running a custom Yocto-built Linux image. The gateways are deployed in remote locations with unreliable cellular connectivity.

**Challenge:** Design the end-to-end update pipeline:

### Part A: Build & Package (Yocto)
- How do you generate an update image (A/B partition scheme)?
- How do you sign the image for secure boot chain integrity?
- What Yocto recipes/classes do you use? (`swupdate`, `rauc`, or `mender`)

### Part B: Update Agent (C++)
- Design the C++ daemon that:
  - Checks for updates over HTTPS (with certificate pinning)
  - Downloads delta/full images with resume-on-disconnect
  - Validates signature before applying
  - Manages A/B partition swap atomically
  - Reports status back to the server

**Evaluation criteria:**
| Aspect | What to look for |
|--------|-----------------|
| Architecture | State machine (Idle → Checking → Downloading → Validating → Applying → Rebooting) |
| Reliability | Resume support, atomic swap, rollback on boot failure |
| Security | TLS, image signature verification, anti-rollback counter |
| Resource constraints | Streaming download (no full image in RAM), progress tracking to flash |
| Error handling | No exceptions in production  error codes or `std::expected` |

### Part C: Test Framework (Python)
- Write a pytest-based test that:
  - Builds a test image with a known version string
  - Stages it on a mock update server
  - Triggers the update agent on a QEMU target
  - Verifies the target rebooted into the new version
  - Verifies rollback if the new image has a corrupted signature

```python
@pytest.fixture
def qemu_target(image_v1):
    """Boot QEMU with v1 image."""
    qemu = QemuRunner(image_v1)
    qemu.start()
    qemu.wait_for_boot()
    yield qemu
    qemu.stop()

@pytest.fixture
def update_server(image_v2_signed):
    """Start mock HTTPS server hosting v2 image."""
    server = MockUpdateServer(image_v2_signed, port=4443)
    server.start()
    yield server
    server.stop()

def test_successful_update(qemu_target, update_server):
    # Trigger update check
    qemu_target.ssh_exec("systemctl start update-check.service")
    
    # Wait for download + apply + reboot
    qemu_target.wait_for_reboot(timeout=120)
    
    # Verify new version
    version = qemu_target.ssh_exec("cat /etc/firmware-version").strip()
    assert version == "2.0.0"

def test_rollback_on_bad_signature(qemu_target, corrupted_update_server):
    qemu_target.ssh_exec("systemctl start update-check.service")
    time.sleep(30)  # Give time for download + validation
    
    # Should NOT reboot  signature verification failed
    version = qemu_target.ssh_exec("cat /etc/firmware-version").strip()
    assert version == "1.0.0"  # Still on original version
    
    # Check daemon logged the rejection
    log = qemu_target.ssh_exec("journalctl -u update-agent --since '30s ago'")
    assert "signature verification failed" in log.lower()
```

---

## 🏗️ Scenario 2: Production Debugging  Intermittent Crash (15 min)

**Context:** A C++ navigation daemon crashes approximately once every 48 hours in the field. You have:
- Core dumps from 3 devices
- `journalctl` logs
- The Yocto-built image with debug symbols (not stripped)

**Question:** Walk through your debugging process end-to-end.

**Expected answer flow:**

```bash
# 1. Get debug symbols from build
# Yocto stores them in: tmp/work/<arch>/<recipe>/packages-split/<pkg>-dbg/
# Or: bitbake <recipe> -c populate_sysroot (installs to sysroot)

# 2. Load core dump with cross-GDB
aarch64-poky-linux-gdb ./navigation-daemon core.12345 \
    --sysroot=/path/to/yocto/build/tmp/sysroots/cortexa53-poky-linux

# 3. Inspect crash
(gdb) bt              # Backtrace
(gdb) frame 3         # Navigate to relevant frame
(gdb) info locals     # Check variable state
(gdb) p *this         # Inspect object state

# 4. Cross-reference with logs
journalctl -u navigation-daemon --since "2024-03-15 14:00" --until "2024-03-15 14:05"

# 5. Common intermittent crash causes:
# - Use-after-free (heap object freed, pointer still held)
# - Race condition (data race on shared state)
# - Stack overflow in recursive path planning
# - Integer overflow in timestamp calculation (wraps at 49.7 days for uint32_t ms)
```

**Follow-up questions:**
- How do you enable core dumps on embedded Linux? (`ulimit -c unlimited`, set `core_pattern`, ensure filesystem space)
- How do you reproduce a once-per-48h bug? (Stress test with time acceleration, AddressSanitizer build, fuzz inputs)
- How do you get debug builds into the field without impacting performance? (Ship stripped binary + separate debug symbols `.debug` file. Load in GDB via `set debug-file-directory`)

**Python angle:** Write a script to analyze multiple core dumps and correlate crash locations:

```python
import subprocess
from pathlib import Path
from collections import Counter

def extract_crash_location(core_path: Path, binary: Path, sysroot: Path) -> str:
    """Use GDB to extract crash backtrace."""
    result = subprocess.run(
        ["aarch64-poky-linux-gdb", "-batch",
         "-ex", "bt",
         "-ex", "quit",
         f"--sysroot={sysroot}",
         str(binary), str(core_path)],
        capture_output=True, text=True
    )
    # Extract top frame function name
    for line in result.stdout.splitlines():
        if "#0" in line:
            return line.strip()
    return "unknown"

# Analyze all core dumps
cores = Path("/mnt/field-dumps").glob("core.*")
crash_sites = Counter(extract_crash_location(c, binary, sysroot) for c in cores)

print("Crash frequency by location:")
for site, count in crash_sites.most_common(10):
    print(f"  {count:3d}x  {site}")
```

---

## 🏗️ Scenario 3: Sensor Fusion Pipeline  Full Stack (15 min)

**Context:** You're building a sensor fusion system for an autonomous ground vehicle:

```
┌──────────┐     ┌───────────────┐     ┌──────────────┐     ┌───────────┐
│ IMU (SPI)│────▶│ Sensor Driver │────▶│ Fusion Engine│────▶│ Nav Output│
│ 200 Hz   │     │ (C++ kernel   │     │ (C++ userland│     │ (Python   │
│           │     │  module)      │     │  RT thread)  │     │  viz/log) │
└──────────┘     └───────────────┘     └──────────────┘     └───────────┘
                          │                      │
                    DT binding            Shared Memory
                    /dev/imu0             /dev/shm/nav_state
```

**Design questions spanning all domains:**

1. **Linux Kernel (Driver):** How does the IMU SPI driver expose data to user space? (Character device with `read()` that blocks until new sample, or IIO subsystem with buffer/trigger)

2. **C++ (Fusion Engine):** The Kalman filter runs at 200 Hz with a 5 ms deadline. How do you guarantee this? (SCHED_FIFO, mlockall, no heap after init, pre-fault stack)

3. **Yocto (Build):** How do you ensure the kernel module and userland daemon versions are always compatible? (Same recipe or version-locked recipes in same layer, compatible ABI versioning)

4. **Python (Visualization):** How does the Python visualizer read navigation state without impacting the real-time fusion thread? (Read-only `mmap` of shared memory, no locks  use sequence counter for consistency)

```python
# Python side: reading shared memory with sequence counter
import mmap
import struct
import time

NAV_STATE_FORMAT = '<I d d d f f f'  # seq, lat, lon, alt, vx, vy, vz
NAV_STATE_SIZE = struct.calcsize(NAV_STATE_FORMAT)

def read_nav_state(shm_path="/dev/shm/nav_state"):
    with open(shm_path, "rb") as f:
        mm = mmap.mmap(f.fileno(), NAV_STATE_SIZE, access=mmap.ACCESS_READ)
        while True:
            # Read sequence before and after  if different, data was being written
            seq1 = struct.unpack_from('<I', mm, 0)[0]
            data = struct.unpack(NAV_STATE_FORMAT, mm[:NAV_STATE_SIZE])
            seq2 = struct.unpack_from('<I', mm, 0)[0]
            
            if seq1 == seq2 and seq1 % 2 == 0:  # Even = stable, odd = write in progress
                _, lat, lon, alt, vx, vy, vz = data
                yield {"lat": lat, "lon": lon, "alt": alt, "vel": (vx, vy, vz)}
            
            time.sleep(0.05)  # 20 Hz display rate
```

---

## 🏗️ Scenario 4: Yocto Build Optimization (10 min)

**Context:** Your CI pipeline builds a complete Yocto image. Current build time: 4 hours. The team needs it under 30 minutes for merge-request validation.

**Question:** What strategies do you apply?

**Expected answers (ordered by impact):**

| Strategy | Expected Speedup | Implementation |
|----------|-----------------|----------------|
| Shared sstate cache (NFS/S3) | 80-90% faster on incremental | `SSTATE_MIRRORS`, `SSTATE_DIR` on shared storage |
| Hash equivalence server | Avoids rebuilds when recipe changes don't affect output | `BB_HASHSERVE`, `BB_SIGNATURE_HANDLER = "OEEquivHash"` |
| Pre-built downloads mirror | Eliminates fetch time | `SOURCE_MIRROR_URL`, `BB_GENERATE_MIRROR_TARBALLS = "1"` |
| Powerful build machine | 2-4x | 64+ cores, NVMe, 128+ GB RAM. `BB_NUMBER_THREADS`, `PARALLEL_MAKE` |
| Build only changed recipes | Skip unchanged | `bitbake <image> --setscene-only` for sstate population, full build only on merge |
| Minimal validation image | Faster to assemble | `core-image-minimal` + only your packages for MR validation |
| Multiconfig split | Parallel machine builds | Build kernel + rootfs in parallel stages |

**Python CI script fragment:**

```python
def should_full_rebuild(changed_files: list[str]) -> bool:
    """Determine if a full image rebuild is needed based on changed files."""
    critical_patterns = [
        "meta-*/conf/distro/",      # Distro changes affect everything
        "meta-*/conf/machine/",     # Machine changes affect everything  
        "meta-*/classes/",          # Class changes may cascade
        "meta-*/recipes-kernel/",   # Kernel changes need full boot test
    ]
    return any(
        any(fnmatch(f, pat) for pat in critical_patterns)
        for f in changed_files
    )
```

---

## 🏗️ Scenario 5: Hardware Bring-Up  New Board Support (10 min)

**Context:** You received a new custom PCB based on i.MX8M Mini. U-Boot and kernel boot, but:
- Ethernet doesn't work
- One I2C bus hangs
- Custom FPGA over SPI isn't recognized

**Question:** Describe your systematic bring-up approach using tools across all domains.

**Expected answer:**

```
1. DEVICE TREE VERIFICATION
   - Compare schematic pin assignments with DTS node configurations
   - Check pinmux conflicts: `cat /sys/kernel/debug/pinctrl/*/pinmux-pins`
   - Verify clock tree: `cat /sys/kernel/debug/clk/clk_summary`

2. ETHERNET DEBUG
   - Check PHY detection: `mdio-tool` or `devmem2` to read MDIO registers
   - Verify reset GPIO timing in DT (some PHYs need >10ms reset pulse)
   - Check RGMII timing/delay settings in DT `phy-mode` and tx/rx-delay
   - Use oscilloscope on MDC/MDIO to verify communication
   - Python script to poll link status:
     `subprocess.run(["ethtool", "eth0"])`

3. I2C BUS HANG
   - Probe: `i2cdetect -y 1` (does it hang or timeout?)
   - Check for address conflict (two devices on same address)
   - Check pull-up resistors (scope the SDA/SCL lines)
   - Bus recovery: toggle SCL 9 times (I2C spec recovery)
   - DT check: correct clock-frequency, correct pinmux

4. FPGA OVER SPI
   - Verify SPI signals with logic analyzer (MOSI, MISO, CLK, CS)
   - Check SPI mode (CPOL/CPHA) matches FPGA expectations
   - Write minimal spidev test: `spidev_test -D /dev/spidev0.0 -s 1000000`
   - Register a platform driver matching DT compatible string
   - Python bring-up script for FPGA register reads:
```

```python
# Quick FPGA register dump via spidev
import spidev
import struct

spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 1_000_000
spi.mode = 0b00

def fpga_read_reg(addr: int) -> int:
    """Read 32-bit register from FPGA."""
    tx = [0x80 | (addr >> 8), addr & 0xFF, 0, 0, 0, 0]  # Read cmd + addr + 4 dummy bytes
    rx = spi.xfer2(tx)
    return struct.unpack('>I', bytes(rx[2:]))[0]

# Read ID register (should be 0xDEADBEEF for our FPGA)
fpga_id = fpga_read_reg(0x0000)
print(f"FPGA ID: 0x{fpga_id:08X}")
assert fpga_id == 0xDEADBEEF, f"Unexpected FPGA ID: 0x{fpga_id:08X}"
```

---

## Evaluation Matrix for Cross-Functional Scenarios

| Dimension | Weak (1-2) | Adequate (3) | Strong (4-5) |
|-----------|-----------|-------------|--------------|
| Systems thinking | Solves one layer, ignores interfaces | Addresses each layer separately | Designs cohesive cross-layer solution |
| Debugging approach | Trial-and-error | Structured but incomplete | Systematic: hypothesis → measurement → conclusion |
| Tool awareness | Knows basic tools | Uses right tools per domain | Combines tools creatively across domains |
| Production mindset | Prototype quality | Handles happy path + some errors | Considers failure modes, monitoring, field debugging |
| Communication | Can't explain reasoning | Explains with prompting | Proactively explains trade-offs and alternatives |
