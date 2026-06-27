# Silicon & Embedded Systems Domain Questions

## FPGA & ASIC Design

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 1 | Compare FPGA vs ASIC. When would you choose each for a product? | SDE 2 | FPGA: reconfigurable, faster time-to-market, lower NRE, good for low-volume/prototyping. ASIC: higher performance, lower power/unit cost at volume, fixed function. Crossover: ~100K-1M units. | What about structured ASICs? How do you estimate FPGA→ASIC cost savings? When does an FPGA make sense even at high volume? |
| 2 | Explain timing closure. What do you do when a design doesn't meet timing? | SDE 3 | Timing closure: all paths meet setup/hold constraints at target frequency. Fixes: pipeline stages (break critical path), retiming, logic restructuring, placement constraints, reduce fan-out, use DSP/BRAM instead of LUT chains. | What's the relationship between clock frequency and power? How does multi-cycle path constraint help? What about false paths? |
| 3 | How do you handle clock domain crossings (CDC)? What are the failure modes? | SDE 3 | Single-bit: 2-FF synchronizer (metastability resolution). Multi-bit: Gray code for counters, async FIFO (dual-clock FIFO with Gray-coded pointers), handshake protocols. Failure: metastability → wrong value → data corruption. | What's MTBF calculation for synchronizers? How do you verify CDC in simulation? What about reset domain crossings? |
| 4 | Describe the FPGA design flow from RTL to bitstream. | SDE 2 | RTL (Verilog/VHDL) → Synthesis (→ netlist) → Implementation (place & route) → Timing analysis (STA) → Bitstream generation → Device programming. Simulation at each stage: behavioral, post-synthesis, post-implementation. | What's the difference between behavioral and gate-level simulation? How do you handle timing violations found post-route? |

## DMA & Memory Architecture

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 5 | Explain DMA. Why is it critical for high-bandwidth embedded systems? | SDE 2 | DMA: peripheral transfers data to/from memory without CPU intervention. CPU sets up transfer (source, dest, length, direction), DMA controller executes, interrupts on completion. Frees CPU for computation during I/O. | What's scatter-gather DMA? How do you handle cache coherency with DMA? What about IOMMU? |
| 6 | How do you handle cache coherency between CPU and DMA in an ARM system? | SDE 3 | Options: cache flush/invalidate before/after DMA, use uncached memory regions (device memory), hardware cache coherent interconnect (ACE/CHI). For Rx: invalidate before DMA starts. For Tx: flush before DMA starts. | What's the performance impact of cache maintenance? When would you use coherent vs non-coherent DMA? |
| 7 | Explain memory-mapped I/O vs port-mapped I/O. How do you prevent compiler optimization issues? | SDE 2 | MMIO: peripherals mapped into address space; accessed like memory. Port-mapped: separate address space with special instructions (x86 IN/OUT). Use `volatile` keyword to prevent compiler reordering/caching reads. Memory barriers for hardware reordering. | What's the difference between compiler barriers and memory barriers? When do you need DSB/DMB/ISB on ARM? |

## Interrupts & RTOS

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 8 | Explain interrupt latency. How do you minimize it? | SDE 2-3 | Interrupt latency = time from IRQ assertion to ISR first instruction. Components: finish current instruction + save context + vector lookup + pipeline flush. Minimize: short ISRs (top-half/bottom-half), proper priority, tail-chaining (Cortex-M), zero-latency interrupts. | What's the worst-case interrupt latency on Cortex-M? How do you measure it? What causes jitter? |
| 9 | What is priority inversion? Explain the Mars Pathfinder incident and the fix. | SDE 2-3 | Priority inversion: high-priority task blocked by low-priority task holding a resource, while medium-priority task preempts the low-priority one. Pathfinder: meteorological task (low) held bus mutex, blocked communications (high). Fix: priority inheritance protocol low-priority task temporarily inherits high priority. | Compare priority inheritance vs priority ceiling. What about in Linux (rt_mutex)? How do you detect priority inversion? |
| 10 | Design an ISR for a UART receive buffer. What are the constraints? | SDE 2 | ISR: read byte from hardware register, store in ring buffer, advance write pointer, clear interrupt flag. Constraints: no blocking, no malloc, minimal time (<10μs typical), volatile pointers, consider buffer overflow (drop or overwrite). Signal application task via semaphore/event flag. | How do you handle buffer overflow? DMA vs interrupt-driven for high baud rates? How does UART FIFO affect ISR frequency? |
| 11 | Compare FreeRTOS vs Zephyr vs Linux RT. When would you choose each? | SDE 2-3 | FreeRTOS: minimal footprint (10KB), MCUs, certifiable (SAFERTOS). Zephyr: richer RTOS, networking stack, multi-arch, growing ecosystem. Linux RT (PREEMPT_RT): full Linux with real-time scheduling, MPUs with ≥32MB RAM, deterministic but higher latency than bare RTOS. | What determinism guarantees does each provide? How do you validate worst-case execution time? What about Xenomai? |

## Watchdog & Reliability

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 12 | Design a watchdog strategy for a safety-critical embedded system. | SDE 3 | Multi-level: (1) HW watchdog reset if SW hangs. (2) SW watchdog monitors individual tasks (task must check in). (3) Window watchdog kick too early OR too late triggers reset. Sequence: each task has its own deadline; supervisor aggregates and kicks HW WDT. | What's the difference between independent and window watchdog? How do you handle watchdog during flash programming? What about multi-core watchdog? |
| 13 | Explain the boot sequence for an embedded Linux device (bootloader chain). | SDE 2-3 | ROM bootloader (fixed, loads SPL from boot media) → SPL/U-Boot SPL (initializes DRAM, loads U-Boot) → U-Boot (loads kernel + DTB + initramfs) → Linux kernel (decompresses, initializes subsystems, mounts rootfs) → init/systemd. | How do you implement secure boot at each stage? What's verified boot vs measured boot? How does A/B booting work for OTA? |

## Low-Power Design

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 14 | How do you design a battery-powered IoT sensor to last 5 years on a coin cell? | SDE 3 | Power budget: CR2032 ~225mAh. At 5 years → avg current ≤5μA. Strategy: deep sleep (nA) with periodic wake; aggressive duty cycling; wake-on-interrupt; choose MCU with <1μA sleep; optimize radio (BLE long range, LoRa for distance); batch transmissions; voltage-aware operation. | How do you measure nA-level currents? What's the energy cost of a BLE advertisement? How do you handle clock drift during long sleep? |
| 15 | Explain power domains and voltage scaling in modern SoCs. | SDE 3 | Power domains: independently switchable power regions (turn off unused peripherals). DVFS (Dynamic Voltage/Frequency Scaling): reduce voltage+frequency when load is low (power ∝ V²×f). Power states: run → idle → standby → shutdown. Retention: keep SRAM powered but clocks off. | How does power gating work at the transistor level? What's the wake-up latency trade-off? How do you handle peripherals across power domains (isolation cells)? |
