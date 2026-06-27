# Yocto for Xilinx Zynq / ZynqMP  Platform Engineering

**Focus: meta-xilinx, FPGA bitstream integration, PetaLinux vs pure Yocto, PS-PL interface, boot flow**

---

## ⚡ Quick-Fire (Zynq + Yocto Gauge)

| Question | Expected Answer |
|----------|----------------|
| What is the difference between Zynq-7000 and ZynqMP? | Zynq-7000: dual Cortex-A9, 32-bit. ZynqMP (UltraScale+): quad A53 + dual R5, 64-bit, PMU firmware. |
| What is FSBL? | First Stage Boot Loader  loads bitstream, initializes PS clocks/DDR, loads U-Boot/ATF. |
| What is PMU firmware on ZynqMP? | Platform Management Unit firmware  power domains, reset, boot sequencing. Runs on MicroBlaze in PMU. |
| What is ATF on ZynqMP? | ARM Trusted Firmware  EL3 runtime, PSCI for power management, secure monitor. |
| What layer provides Zynq support? | `meta-xilinx` (meta-xilinx-core, meta-xilinx-bsp, meta-xilinx-tools) |
| What is `HDF` / `XSA`? | Hardware Definition File  exported from Vivado. Contains PS config, bitstream, device tree fragment. |
| PetaLinux vs pure Yocto? | PetaLinux: Xilinx wrapper around Yocto with templates/scripts. Pure Yocto: more control, no vendor lock. |
| What is `MACHINE` for Zynq? | e.g., `zcu102-zynqmp`, `zc706-zynq7`, or custom `my-board-zynqmp` |
| How does bitstream get loaded? | FSBL loads during boot (early), or Linux `fpga_manager` loads at runtime from `/lib/firmware/` |
| What is device tree overlay for PL? | Describes PL peripherals (AXI IPs). Applied at runtime when bitstream is loaded. |

---

## 🔬 Q1: Custom Zynq Board BSP in Yocto

**Question:** You designed a custom ZynqMP board. Walk through creating
the Yocto BSP from scratch.

**Expected answer:**

```
meta-custom-board/
├── conf/
│   ├── layer.conf
│   └── machine/
│       └── custom-zynqmp.conf
├── recipes-bsp/
│   ├── fsbl/
│   │   └── fsbl-firmware_%.bbappend
│   ├── pmu-firmware/
│   │   └── pmu-firmware_%.bbappend
│   ├── arm-trusted-firmware/
│   │   └── arm-trusted-firmware_%.bbappend
│   ├── u-boot/
│   │   └── u-boot-xlnx_%.bbappend
│   └── device-tree/
│       ├── device-tree_%.bbappend
│       └── files/
│           └── custom-board.dtsi
├── recipes-kernel/
│   └── linux-xlnx/
│       └── linux-xlnx_%.bbappend
└── recipes-firmware/
    └── fpga-bitstream/
        └── fpga-bitstream.bb
```


**Machine configuration:**

```bitbake
# conf/machine/custom-zynqmp.conf
#@TYPE: Machine
#@NAME: Custom ZynqMP Industrial Controller
#@DESCRIPTION: Custom board based on ZU9EG

require conf/machine/zynqmp-generic.conf

# Or define from scratch:
SOC_VARIANT = "eg"
MACHINEOVERRIDES =. "custom-zynqmp:"

MACHINE_FEATURES = "rtc usbhost vfat ext4"

# Boot components
SPL_BINARY = ""
UBOOT_MACHINE = "xilinx_zynqmp_custom_defconfig"
UBOOT_ENTRYPOINT = "0x00200000"
UBOOT_LOADADDRESS = "0x00200000"

# FPGA bitstream
HDF_MACHINE = "custom-zynqmp"
HDF_FILE = "custom_platform.xsa"
# XSA from Vivado lives in:
# meta-custom-board/recipes-bsp/hdf/files/custom_platform.xsa

# Kernel
PREFERRED_PROVIDER_virtual/kernel = "linux-xlnx"
KERNEL_DEVICETREE = "xilinx/custom-zynqmp.dtb"

# Image
IMAGE_BOOT_FILES = "BOOT.BIN boot.scr Image custom-zynqmp.dtb"

# Serial console
SERIAL_CONSOLES = "115200;ttyPS0"
```

**Follow-up probes:**
- How do you handle XSA changes from FPGA team? (Version-controlled XSA in separate recipe, CI triggers rebuild of FSBL + device-tree)
- What if PS DDR config changes? (Requires FSBL rebuild  XSA contains DDR training params)
- How do you automate Vivado → Yocto pipeline? (CI job: export XSA from Vivado project, commit to BSP layer, trigger Yocto build)

---

## 🔬 Q2: FPGA Bitstream Integration and Runtime Loading

**Question:** Your system needs to load different FPGA configurations
at runtime (different operational modes). Design the Yocto recipe
and Linux userland approach.

**Expected answer:**

```bitbake
# recipes-firmware/fpga-bitstream/fpga-bitstream.bb
SUMMARY = "FPGA bitstreams for runtime loading"
LICENSE = "CLOSED"

SRC_URI = " \
    file://mode_normal.bit.bin \
    file://mode_diagnostics.bit.bin \
    file://mode_calibration.bit.bin \
    file://pl-normal.dtbo \
    file://pl-diagnostics.dtbo \
"

do_install() {
    install -d ${D}/lib/firmware
    install -m 0644 ${WORKDIR}/*.bit.bin ${D}/lib/firmware/

    install -d ${D}/lib/firmware/overlays
    install -m 0644 ${WORKDIR}/*.dtbo ${D}/lib/firmware/overlays/
}

FILES:${PN} = "/lib/firmware/"
```

**Linux runtime loading from C++:**

```cpp
#include <filesystem>
#include <fstream>

class FpgaManager {
public:
    enum class Mode { Normal, Diagnostics, Calibration };

    std::expected<void, std::string> load_bitstream(Mode mode) {
        const auto bitstream = bitstream_path(mode);
        const auto overlay = overlay_path(mode);

        // 1. Remove current overlay if loaded
        if (current_overlay_loaded_) {
            remove_overlay();
        }

        // 2. Load bitstream via fpga_manager sysfs
        std::ofstream flags("/sys/class/fpga_manager/fpga0/flags");
        flags << "0";  // Full reconfiguration
        flags.close();

        std::ofstream firmware("/sys/class/fpga_manager/fpga0/firmware");
        firmware << bitstream.filename().string();
        firmware.close();

        // 3. Apply device tree overlay for PL peripherals
        if (!apply_overlay(overlay)) {
            return std::unexpected("Failed to apply DT overlay");
        }

        current_mode_ = mode;
        current_overlay_loaded_ = true;
        return {};
    }

private:
    std::filesystem::path bitstream_path(Mode m) const {
        switch (m) {
            case Mode::Normal: return "/lib/firmware/mode_normal.bit.bin";
            case Mode::Diagnostics: return "/lib/firmware/mode_diagnostics.bit.bin";
            case Mode::Calibration: return "/lib/firmware/mode_calibration.bit.bin";
        }
    }

    bool apply_overlay(const std::filesystem::path& dtbo) {
        // Use configfs dt overlay interface
        auto overlay_dir = "/sys/kernel/config/device-tree/overlays/pl";
        std::filesystem::create_directory(overlay_dir);
        std::ofstream(std::string(overlay_dir) + "/path") << dtbo.filename().string();
        return std::filesystem::exists(std::string(overlay_dir) + "/status");
    }

    void remove_overlay() {
        std::filesystem::remove("/sys/kernel/config/device-tree/overlays/pl");
        current_overlay_loaded_ = false;
    }

    Mode current_mode_ = Mode::Normal;
    bool current_overlay_loaded_ = false;
};
```


**Follow-up probes:**
- What is `.bit` vs `.bit.bin`? (`.bit` has Xilinx header. `.bit.bin` is raw for fpga_manager. Convert with `bootgen` or Vivado `write_bitstream -bin_file`)
- What happens to AXI peripherals during reconfiguration? (All PL-side AXI becomes invalid. Must remove driver/overlay first, then reload.)
- How do you verify the bitstream version matches software? (Version register in PL at known AXI offset. Read after load, compare against expected.)

---

## 🔬 Q3: ZynqMP Boot Flow  Complete Chain

**Question:** Trace the ZynqMP boot from power-on to Linux userspace.
Include all firmware components and where Yocto builds them.

**Expected answer:**

```
Power-On
    │
    ▼
┌────────────────────────────────────────┐
│ 1. CSU ROM (Chip Security Unit)        │  ← Silicon, not buildable
│    - Reads boot mode pins              │
│    - Loads PMU firmware from boot media│
│    - Authenticates if Secure Boot      │
└────────────────┬───────────────────────┘
                 ▼
┌────────────────────────────────────────┐
│ 2. PMU Firmware (MicroBlaze)           │  ← Yocto: pmu-firmware recipe
│    - Power domain management           │     (from meta-xilinx-tools or
│    - Clock configuration               │      Xilinx SDK / Vitis)
│    - Runs throughout system lifetime   │
└────────────────┬───────────────────────┘
                 ▼
┌────────────────────────────────────────┐
│ 3. FSBL (First Stage Boot Loader)      │  ← Yocto: fsbl-firmware recipe
│    - Initializes DDR (training)        │     (built from XSA/HDF)
│    - Loads PL bitstream (optional)     │
│    - Loads ATF + U-Boot to DDR         │
│    - Hands off to ATF at EL3           │
└────────────────┬───────────────────────┘
                 ▼
┌────────────────────────────────────────┐
│ 4. ATF (ARM Trusted Firmware) - EL3    │  ← Yocto: arm-trusted-firmware recipe
│    - PSCI (power state management)     │
│    - Secure monitor                    │
│    - Drops to U-Boot at EL2            │
└────────────────┬───────────────────────┘
                 ▼
┌────────────────────────────────────────┐
│ 5. U-Boot - EL2 (or EL1 if no ATF)    │  ← Yocto: u-boot-xlnx recipe
│    - Boot menu, env, network boot      │
│    - Loads kernel Image + DTB + initrd │
│    - bootm / booti                     │
└────────────────┬───────────────────────┘
                 ▼
┌────────────────────────────────────────┐
│ 6. Linux Kernel - EL1                  │  ← Yocto: linux-xlnx recipe
│    - Parses DTB                        │
│    - Probes drivers (PS + PL)          │
│    - Mounts rootfs                     │
│    - Executes /sbin/init (systemd)     │
└────────────────┬───────────────────────┘
                 ▼
┌────────────────────────────────────────┐
│ 7. User Space (systemd)                │  ← Yocto: your image recipe
│    - Services start                    │
│    - Application launches              │
└────────────────────────────────────────┘
```

**All components packaged into BOOT.BIN:**

```bitbake
# Created by bootgen tool from boot.bif:
# BOOT.BIN = PMU_FW + FSBL + ATF + U-Boot + [Bitstream]
# Yocto class: xilinx-bootbin handles this via do_deploy
```

**Follow-up probes:**
- What is a `.bif` file? (Boot Image Format descriptor  tells `bootgen` how to package BOOT.BIN)
- How do you debug a hang between FSBL and U-Boot? (JTAG + Vivado HW Manager, or FSBL debug prints via UART)
- What is the R5 used for in ZynqMP? (Real-time processing  bare-metal or FreeRTOS. Loaded by ATF or remoteproc from Linux)

---

## 🔬 Q4: PS-PL Communication  AXI/UIO Architecture

**Question:** Your FPGA has 4 custom AXI-Lite peripherals:
- Motor controller (registers for position, velocity, status)
- ADC capture (DMA ring buffer)
- Digital I/O (32 GPIO pins)
- Timing engine (trigger sequencer)

Design the Linux driver strategy and C++ access layer.

**Expected answer:**

```
Strategy: UIO for simple register access, custom DMA driver for ADC

┌─────────────────────────────────────────────────────┐
│ User Space (C++ Application)                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐│
│  │MotorCtrl │ │ AdcCapture│ │ DigitalIO │ │Timing  ││
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └───┬────┘│
├───────┼─────────────┼────────────┼────────────┼─────┤
│       │    mmap     │   DMA API  │    mmap    │mmap │
│  /dev/uio0    /dev/adc0    /dev/uio1     /dev/uio2  │
├───────┼─────────────┼────────────┼────────────┼─────┤
│ Kernel│             │            │            │     │
│  UIO driver   Custom DMA     UIO driver   UIO      │
│  (generic)    char driver    (generic)    driver    │
└───────┼─────────────┼────────────┼────────────┼─────┘
        │             │            │            │
    AXI-Lite      AXI-DMA      AXI-Lite    AXI-Lite
        │             │            │            │
┌───────┴─────────────┴────────────┴────────────┴─────┐
│                    FPGA Fabric (PL)                    │
└──────────────────────────────────────────────────────┘
```


**C++ UIO access class:**

```cpp
class UioDevice {
public:
    explicit UioDevice(const std::string& device_path, std::size_t map_size)
        : map_size_(map_size) {
        fd_ = ::open(device_path.c_str(), O_RDWR | O_SYNC);
        if (fd_ < 0) throw std::system_error(errno, std::system_category());

        base_ = static_cast<volatile uint32_t*>(
            ::mmap(nullptr, map_size_, PROT_READ | PROT_WRITE, MAP_SHARED, fd_, 0)
        );
        if (base_ == MAP_FAILED) {
            ::close(fd_);
            throw std::system_error(errno, std::system_category());
        }
    }

    ~UioDevice() noexcept {
        if (base_ && base_ != MAP_FAILED) ::munmap((void*)base_, map_size_);
        if (fd_ >= 0) ::close(fd_);
    }

    uint32_t read_reg(std::size_t offset) const noexcept {
        return base_[offset / sizeof(uint32_t)];
    }

    void write_reg(std::size_t offset, uint32_t value) noexcept {
        base_[offset / sizeof(uint32_t)] = value;
    }

    // Wait for interrupt from PL
    bool wait_interrupt(std::chrono::milliseconds timeout) {
        uint32_t enable = 1;
        ::write(fd_, &enable, sizeof(enable));  // Re-enable IRQ

        struct pollfd pfd = {fd_, POLLIN, 0};
        int ret = ::poll(&pfd, 1, timeout.count());
        if (ret > 0) {
            uint32_t count;
            ::read(fd_, &count, sizeof(count));  // Acknowledge
            return true;
        }
        return false;
    }

    // Non-copyable, movable
    UioDevice(const UioDevice&) = delete;
    UioDevice& operator=(const UioDevice&) = delete;
    UioDevice(UioDevice&& other) noexcept
        : fd_(std::exchange(other.fd_, -1)),
          base_(std::exchange(other.base_, nullptr)),
          map_size_(other.map_size_) {}

private:
    int fd_ = -1;
    volatile uint32_t* base_ = nullptr;
    std::size_t map_size_;
};

// Typed motor controller built on top
class MotorController {
public:
    explicit MotorController(const std::string& uio = "/dev/uio0")
        : uio_(uio, 4096) {
        // Verify IP core version
        uint32_t version = uio_.read_reg(REG_VERSION);
        if (version < MIN_VERSION)
            throw std::runtime_error("FPGA motor IP too old");
    }

    void set_target_position(int32_t counts) {
        uio_.write_reg(REG_TARGET_POS, static_cast<uint32_t>(counts));
    }

    int32_t get_actual_position() const {
        return static_cast<int32_t>(uio_.read_reg(REG_ACTUAL_POS));
    }

    bool motion_complete() const {
        return (uio_.read_reg(REG_STATUS) & STATUS_DONE) != 0;
    }

    bool wait_done(std::chrono::milliseconds timeout) {
        return uio_.wait_interrupt(timeout);
    }

private:
    static constexpr std::size_t REG_VERSION    = 0x00;
    static constexpr std::size_t REG_TARGET_POS = 0x04;
    static constexpr std::size_t REG_ACTUAL_POS = 0x08;
    static constexpr std::size_t REG_STATUS     = 0x0C;
    static constexpr uint32_t STATUS_DONE       = 0x01;
    static constexpr uint32_t MIN_VERSION       = 0x0200;

    UioDevice uio_;
};
```

**Device Tree for UIO:**

```dts
motor_controller: motor@a0000000 {
    compatible = "generic-uio";
    reg = <0x0 0xa0000000 0x0 0x1000>;
    interrupt-parent = <&gic>;
    interrupts = <0 89 4>;  /* SPI 89, level high */
};
```

---

## 🔬 Q5: Zynq Multicore  Linux + FreeRTOS on R5

**Question:** Your ZynqMP runs Linux on A53 and FreeRTOS on R5 for
real-time motor control. How do you:
1. Build both images in a single Yocto project?
2. Communicate between A53 (Linux) and R5 (FreeRTOS)?
3. Load/start the R5 firmware from Linux?

**Expected answer:**

```bitbake
# 1. Multiconfig build for both A53 and R5
# conf/multiconfig/a53-linux.conf
MACHINE = "custom-zynqmp"
DISTRO = "company-linux"
TCLIBC = "glibc"

# conf/multiconfig/r5-freertos.conf (or baremetal)
MACHINE = "custom-zynqmp-r5"
DISTRO = "xilinx-freertos"
TCLIBC = "newlib"
```

```bitbake
# Build both:
# bitbake mc:a53-linux:platform-image mc:r5-freertos:motor-firmware
```

**Communication: OpenAMP / RPMsg:**

```
┌───────────────────────┐       ┌──────────────────────┐
│   A53 (Linux)          │       │   R5 (FreeRTOS)      │
│                        │       │                      │
│  /dev/rpmsg_ctrl0      │       │  rpmsg_endpoint      │
│       ↕                │       │       ↕              │
│  remoteproc driver     │  ←→   │  OpenAMP library     │
│       ↕                │       │       ↕              │
│  Shared Memory (DDR)   │◄─────►│  Shared Memory       │
│  + Vring buffers       │       │  + Vring buffers     │
└───────────────────────┘       └──────────────────────┘
    IPI (Inter-Processor Interrupt) for doorbell
```

**Linux side (C++):**

```cpp
// Load and start R5 firmware via remoteproc
void start_r5_firmware() {
    // Stop if already running
    write_sysfs("/sys/class/remoteproc/remoteproc0/state", "stop");

    // Set firmware path
    write_sysfs("/sys/class/remoteproc/remoteproc0/firmware", "motor-firmware.elf");

    // Start
    write_sysfs("/sys/class/remoteproc/remoteproc0/state", "start");
}

// Communicate via RPMsg
class RpmsgChannel {
public:
    RpmsgChannel(const std::string& device = "/dev/rpmsg_ctrl0") {
        // Create endpoint
        fd_ = open(device.c_str(), O_RDWR);
        // ... ioctl to create endpoint
    }

    void send(std::span<const uint8_t> data) {
        ::write(fd_, data.data(), data.size());
    }

    std::vector<uint8_t> receive(std::chrono::milliseconds timeout) {
        // poll + read
    }
private:
    int fd_ = -1;
};
```

---

## ⚡ Zynq-Specific Quick Checks

| Question | Expected Answer |
|----------|----------------|
| What is `bootgen`? | Xilinx tool to create BOOT.BIN from .bif descriptor. Packages FSBL + bitstream + ATF + U-Boot. |
| What is `QSPI` boot vs `SD` boot? | QSPI: boots from flash (production). SD: boots from card (development). Set via boot mode pins. |
| What is AXI interconnect? | ARM AMBA bus connecting PS to PL IPs. AXI-Lite for registers, AXI-Full/Stream for DMA. |
| What is the PS-PL clock domain? | PL clock is independent of PS. Crossing requires CDC (Clock Domain Crossing) in FPGA logic. |
| How do you debug PL from Linux? | ILA (Integrated Logic Analyzer) in bitstream + Vivado HW Manager via JTAG. Or: read debug registers via UIO. |
| What is `DT overlay` for PL? | Adds PL peripheral nodes to running device tree. Applied after bitstream load via configfs. |
| What is Vitis vs Vivado? | Vivado: FPGA design (PL). Vitis: Software (PS firmware, embeddings). Vitis uses SDK from XSA. |
