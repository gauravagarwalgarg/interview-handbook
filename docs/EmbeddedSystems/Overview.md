# Embedded Systems & Linux Interview Questions

**Focus: MCU vs MPU, Interrupt Handling, RTOS, Linux Boot Process, Device Drivers, Memory Mapping, IPC**

---

## ⚡ Quick-Fire (Language/Platform Gauge)

| Question | Expected Answer |
|----------|----------------|
| Difference between MCU and MPU? | MCU: integrated Flash/RAM/peripherals, single-chip. MPU: needs external memory, runs OS, has MMU. |
| What does an MMU do? | Virtual-to-physical address translation, memory protection (page faults), isolation between processes |
| What is DMA and why use it? | Direct Memory Access  peripheral transfers data without CPU intervention. Frees CPU for computation. |
| What is a watchdog timer? | Hardware timer that resets the system if not periodically "kicked"  detects hung firmware |
| What is the difference between polling and interrupt-driven I/O? | Polling wastes CPU cycles. Interrupts are event-driven but have latency and context-switch cost. |
| What is interrupt latency? | Time from interrupt assertion to first instruction of ISR. Includes pipeline flush, context save, vector fetch. |
| Can you call `printf` in an ISR? | No. `printf` is non-reentrant (uses heap, locks, buffered I/O). Use a ring buffer + deferred print. |
| What is a priority inversion? | Low-priority task holds a lock needed by high-priority task. Medium-priority task preempts low, blocking high indefinitely. |
| How does priority inheritance solve it? | Temporarily elevate low-priority task to high's priority while holding the lock. |
| What is JTAG/SWD? | Debug interfaces for MCUs. JTAG: 4-5 wire, boundary scan. SWD: 2-wire ARM-specific, lower pin count. |

---

## 🔬 Q1: Interrupt Handling  Nested Vectored Interrupt Controller (NVIC)

**Question:** On an ARM Cortex-M4, you have:
- A 1 kHz timer interrupt sampling an ADC
- A UART RX interrupt receiving GPS data
- A fault handler for hard faults

Design the interrupt priority scheme. What happens if the UART interrupt fires during the ADC ISR?

**Expected discussion:**

```
Priority (lower number = higher priority on Cortex-M):
  0: HardFault (always highest, non-configurable)
  1: ADC Timer ISR (time-critical, must not be delayed)
  3: UART RX ISR (can tolerate ~100μs delay)
```

- **Preemption:** If UART has lower priority than ADC timer, UART is deferred (tail-chained after ADC ISR completes).
- **If UART has higher priority:** It preempts the ADC ISR (nested interrupt). ADC ISR resumes after UART completes.
- **Key constraint:** ISRs must be short. Defer heavy processing to a task/thread via a flag or queue.

**Follow-up:** What is interrupt jitter and how do you measure it? (Oscilloscope on a GPIO toggled in ISR; jitter = max latency - min latency)

---

## 🔬 Q2: RTOS Fundamentals  FreeRTOS Task Design

**Question:** Design the task architecture for a battery management system (BMS) with:
- Cell voltage sampling (10 Hz)
- Temperature monitoring (1 Hz)
- CAN bus communication (event-driven)
- Coulomb counting (100 Hz, highest priority)
- State-of-charge display update (0.5 Hz, lowest priority)

**Expected answer:**

| Task | Priority | Period | Stack Size | Notes |
|------|----------|--------|-----------|-------|
| CoulombCounting | Highest (5) | 10 ms | 256 words | Time-critical, minimal processing |
| CellVoltageSample | High (4) | 100 ms | 512 words | ADC reads, averaging |
| CanComm | Medium (3) | Event-driven | 1024 words | Message parsing, state machine |
| TempMonitor | Low (2) | 1000 ms | 256 words | Simple I2C read |
| DisplayUpdate | Lowest (1) | 2000 ms | 512 words | Can be starved without harm |

**Inter-task communication:**
- CoulombCounting → CanComm: `xQueueSend()` with latest SoC value
- CellVoltageSample → CanComm: Shared structure protected by mutex
- ISR → Task: `xSemaphoreGiveFromISR()` (binary semaphore as event flag)

**Follow-up probes:**
- What happens if CoulombCounting overruns its 10 ms deadline? (Rate monotonic analysis, WCET)
- Stack overflow detection? (`configCHECK_FOR_STACK_OVERFLOW`, watermark analysis)
- Why not use `vTaskDelay` for precise timing? (Use hardware timer + `xTaskNotifyFromISR` instead)

---

## 🔬 Q3: Linux Boot Process  From Power-On to User Space

**Question:** Trace the complete boot sequence of an embedded Linux system on an i.MX8M (ARM Cortex-A53):

**Expected answer:**

```
1. ROM Bootloader (SoC-internal)
   ├── Reads fuse configuration (boot device: eMMC, SD, USB)
   ├── Loads SPL/MLO from boot media
   └── Verifies signature (if Secure Boot / HAB enabled)

2. SPL (Secondary Program Loader) / U-Boot SPL
   ├── Initializes DRAM controller (DDR4 training)
   ├── Sets up minimal clocks
   └── Loads full U-Boot into DRAM

3. U-Boot
   ├── Full hardware init (Ethernet, USB, display)
   ├── Reads environment variables (boot device, kernel args)
   ├── Loads kernel image (zImage/Image) + DTB + initramfs
   ├── Passes control to kernel: bootz <kernel_addr> <initrd_addr> <dtb_addr>
   └── Sets bootargs: "console=ttymxc0,115200 root=/dev/mmcblk0p2 rootfstype=ext4"

4. Linux Kernel
   ├── Decompresses itself (if zImage)
   ├── Parses Device Tree Blob (DTB)
   ├── Initializes memory subsystem, scheduler, interrupts
   ├── Probes device drivers (matching DTB nodes)
   ├── Mounts initramfs (if provided)
   └── Executes /sbin/init (PID 1)

5. Init System (systemd / BusyBox init)
   ├── Mounts filesystems (/proc, /sys, /dev)
   ├── Starts services in dependency order
   └── Launches application
```

**Follow-up probes:**
- What is the Device Tree and why does ARM Linux use it? (Hardware description, replaces board files, runtime-parsed by kernel)
- How do you debug a boot hang between U-Boot and kernel? (Early printk, `earlyprintk` bootarg, JTAG)
- What is Secure Boot / HAB? (High Assurance Boot on NXP  cryptographic chain of trust from ROM to kernel)

---

## 🔬 Q4: Linux Device Drivers  Character Device

**Question:** Write a minimal Linux kernel module that exposes a character device `/dev/sensor0`. The device:
- Returns the last temperature reading on `read()`
- Accepts a calibration offset on `write()`
- Supports `ioctl` for resetting calibration

**Expected answer structure:**

```c
#include <linux/module.h>
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

static int temperature_raw = 2500; // milli-degrees
static int calibration_offset = 0;

static ssize_t sensor_read(struct file *f, char __user *buf, size_t len, loff_t *off) {
    int calibrated = temperature_raw + calibration_offset;
    if (len < sizeof(calibrated)) return -EINVAL;
    if (copy_to_user(buf, &calibrated, sizeof(calibrated))) return -EFAULT;
    return sizeof(calibrated);
}

static ssize_t sensor_write(struct file *f, const char __user *buf, size_t len, loff_t *off) {
    if (len < sizeof(calibration_offset)) return -EINVAL;
    if (copy_from_user(&calibration_offset, buf, sizeof(calibration_offset))) return -EFAULT;
    return sizeof(calibration_offset);
}

static long sensor_ioctl(struct file *f, unsigned int cmd, unsigned long arg) {
    switch (cmd) {
        case 0x01: // IOCTL_RESET_CAL
            calibration_offset = 0;
            return 0;
        default:
            return -ENOTTY;
    }
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .read = sensor_read,
    .write = sensor_write,
    .unlocked_ioctl = sensor_ioctl,
};
```

**Follow-up probes:**
- What is `copy_to_user` / `copy_from_user` and why can't you use `memcpy`? (User-space addresses may be paged out; these functions handle page faults gracefully)
- How do you handle concurrent access? (mutex, spinlock, or atomic depending on context)
- What is the difference between a platform driver and a character driver? (Platform driver binds to Device Tree nodes; char driver exposes user-space file interface)

---

## 🔬 Q5: IPC Mechanisms on Embedded Linux

**Question:** Your system has three processes:
1. **Sensor daemon** (C++, real-time priority, collects IMU data at 200 Hz)
2. **Navigation engine** (C++, consumes sensor data, outputs position)
3. **Ground link** (Python, forwards position to base station over TCP)

Design the IPC architecture. Consider latency, throughput, and crash isolation.

**Expected answer:**

| IPC Mechanism | Between | Rationale |
|---------------|---------|-----------|
| Shared memory + eventfd | Sensor → Navigation | Lowest latency, zero-copy. `eventfd` for notification without polling. |
| Unix domain socket | Navigation → Ground link | Stream-oriented, handles Python's GIL gracefully, crash-isolated |
| POSIX message queue | Alternative to shmem | Kernel-buffered, simpler API, but copies data |

**Shared memory design:**

```
┌─────────────────────────────────────────┐
│  /dev/shm/imu_buffer                     │
│  ┌───────────────────────────────────┐  │
│  │ Header: write_index (atomic), seq  │  │
│  ├───────────────────────────────────┤  │
│  │ Ring buffer: ImuSample[256]       │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

- Producer: writes sample, increments atomic index, signals `eventfd`
- Consumer: reads from last-consumed index to current write index
- Overflow policy: overwrite oldest (consumer detects via sequence number gap)

**Follow-up probes:**
- Why not use pipes? (Copies data, 64 KB default buffer, not suitable for structured high-rate data)
- How do you ensure the shared memory layout is identical between C++ and Python? (Use `struct.pack`/`ctypes` in Python matching C++ `packed` struct layout)
- What about D-Bus? (Too heavy for real-time sensor data  fine for configuration/command messages)

---

## 🔬 Q6: Memory-Mapped I/O from User Space

**Question:** How do you access a hardware register at physical address `0x0209'C000` from a Linux user-space application?

**Expected answer:**

```c
#include <fcntl.h>
#include <sys/mman.h>

int fd = open("/dev/mem", O_RDWR | O_SYNC);
volatile uint32_t *regs = (uint32_t *)mmap(
    NULL, 4096,
    PROT_READ | PROT_WRITE,
    MAP_SHARED,
    fd, 0x0209C000
);

// Read status register (offset 0x00)
uint32_t status = regs[0];

// Write control register (offset 0x04)
regs[1] = 0x0000'0001;

munmap((void*)regs, 4096);
close(fd);
```

**Key discussion:**
- `O_SYNC` ensures uncached access (otherwise CPU cache may serve stale data)
- `/dev/mem` requires root  production should use a proper kernel driver with `ioremap()`
- UIO (User-space I/O) framework is the proper way to do user-space drivers
- Security: `/dev/mem` is often restricted via kernel CONFIG or SELinux

---

## ⚡ Additional Quick Checks

| Question | Key Answer |
|----------|-----------|
| What is `/proc` vs `/sys`? | `/proc`: process info + kernel tuning. `/sys`: device/driver model (kobjects). |
| What is a Device Tree overlay? | Runtime modification of the base DTB (e.g., enable a peripheral on a cape/HAT) |
| What is `mmap` vs `read`/`write` for device access? | `mmap`: zero-copy, direct access. `read`/`write`: buffered, kernel-mediated, safer. |
| What is the difference between hard IRQ and soft IRQ? | Hard: hardware interrupt, top-half. Soft: deferred work, bottom-half (tasklet, workqueue). |
| What is `ioremap` vs `ioremap_nocache`? | `ioremap` maps physical to virtual in kernel. `_nocache` disables caching (required for MMIO). |
| What is kernel preemption? | Kernel can be preempted during execution (CONFIG_PREEMPT). Required for real-time. |
