---
hide:
  - navigation
  - toc
---

# Interview Question Bank

<div class="grid cards" markdown>

-   :material-language-cpp:{ .lg .middle } **C++ (Modern C++14/17/20)**

    ---

    Memory management, RAII, smart pointers, concurrency, design patterns, constrained environments, system architecture

    [:octicons-arrow-right-24: Go to C++ section](C++/Overview.md)

-   :material-chip:{ .lg .middle } **Embedded Systems & Linux**

    ---

    MCU vs MPU, interrupts, RTOS, Linux boot, device drivers, IPC, CAN/SPI/I2C/UART, REST/MQTT

    [:octicons-arrow-right-24: Go to Embedded section](EmbeddedSystems/Overview.md)

-   :material-layers-triple:{ .lg .middle } **Yocto Project**

    ---

    Recipes, layers, BitBake, image customization, SDK, Zynq/ZynqMP, FPGA bitstream integration

    [:octicons-arrow-right-24: Go to Yocto section](Yocto/Overview.md)

-   :material-pipe:{ .lg .middle } **CI/CD & Cloud**

    ---

    GitLab pipelines, Docker builds, SonarQube, quality gates, fleet management, OTA deployment

    [:octicons-arrow-right-24: Go to CI/CD section](CICD_Cloud/Overview.md)

-   :material-language-python:{ .lg .middle } **Python Scripting**

    ---

    Automation, log parsing, serial communication, pytest, HW-in-the-loop testing

    [:octicons-arrow-right-24: Go to Python section](Python/Overview.md)

-   :material-transit-connection-variant:{ .lg .middle } **Cross-Functional Scenarios**

    ---

    End-to-end problems spanning FPGA → Linux → C++ → Cloud → OTA

    [:octicons-arrow-right-24: Go to Scenarios](CrossFunctional/Overview.md)

</div>

---

## Interview Flow

### Round 1  Platform & C++ Deep Dive (60 min)

| Time | Phase | Focus |
|------|-------|-------|
| 0–5 min | Quick-fire + Tricky Output Qs | C/C++ language depth gauge |
| 5–20 min | C++ Architecture + CMake | Modern C++, build systems, API design |
| 20–35 min | Protocols + Embedded Linux | CAN/SPI/I2C, REST/MQTT, IPC, drivers |
| 35–50 min | Yocto Platform | Layers, recipes, image customization, SDK |
| 50–58 min | Design Pattern Exercise | State machine / Observer / Factory |
| 58–60 min | Candidate Questions | |

### Round 2  Delivery, Quality & Systems Integration (60 min)

| Time | Phase | Focus |
|------|-------|-------|
| 0–5 min | Quick-fire (CI/CD + Quality) | Pipeline fluency, SonarQube, coverage |
| 5–20 min | GitLab CI/CD Pipeline Design | Quality gates, containerized builds |
| 20–30 min | Testing Strategy | GTest, pytest, HW-in-loop, coverage |
| 30–45 min | Cross-Functional Scenario | End-to-end: Yocto → CI → OTA |
| 45–55 min | Agile & Software Metrics | Sprint execution, engineering productivity |
| 55–60 min | Candidate Questions | |

---

## Legend

| Marker | Meaning |
|--------|---------|
| ⚡ | Quick-fire  30 seconds expected |
| 🔬 | Deep-dive  5–10 min discussion |
| 🏗️ | Design exercise  10–20 min whiteboard |

---

[:material-chart-box: **Evaluation Metrics Sheet →**](Metrics/EvaluationSheet.md){ .md-button .md-button--primary }

---

## Rubrics & Calibration

Enterprise-grade interviewer rubrics for consistent, fair hiring decisions.

| Resource | Description |
|----------|-------------|
| [:material-scale-balance: **Level Calibration Matrix**](rubrics/level-calibration.md) | SDE 1→4 expectations, signals, red flags |
| [:material-code-braces: **Coding Round Rubric**](rubrics/coding-round.md) | Algorithm & problem-solving scorecard with examples |
| [:material-draw: **LLD Round Rubric**](rubrics/lld-round.md) | Low-level design, OOP, concurrency scorecard |
| [:material-cloud-outline: **HLD Round Rubric**](rubrics/hld-round.md) | System design (Cloud + Embedded) scorecard |
| [:material-pipe: **Infra & CI/CD Rubric**](rubrics/infra-cicd-round.md) | DevOps, observability, deployment scorecard |
| [:material-file-document-edit: **Feedback Template**](rubrics/feedback-template.md) | Copy-paste post-interview writeup |

**Domain Question Banks:**
[Backend](rubrics/domain-questions/backend.md) ·
[Frontend](rubrics/domain-questions/frontend.md) ·
[Networking & OS](rubrics/domain-questions/networking-os.md) ·
[Silicon & Embedded](rubrics/domain-questions/silicon-embedded.md)

[:material-book-open-variant: **All Rubrics →**](rubrics/README.md){ .md-button }


---

## 📋 Question Banks (25+ Questions Each, With Solutions)

### Language-Specific Coding (Output/Gotchas)

| Language | Focus | Link |
|----------|-------|------|
| C++ | Virtual dispatch, move semantics, UB, templates, RAII | [cpp-coding.md](questions/cpp-coding.md) |
| Python | Mutable defaults, GIL, closures, async, metaclasses | [python-coding.md](questions/python-coding.md) |
| Go | Goroutine leaks, channels, nil interface, slice traps | [golang-coding.md](questions/golang-coding.md) |
| Java | String pool, generics erasure, concurrency, streams | [java-coding.md](questions/java-coding.md) |
| C | Pointer arithmetic, struct padding, volatile, MMIO | [c-coding.md](questions/c-coding.md) |

### Round-Specific Question Banks

| Round | Questions | Link |
|-------|-----------|------|
| DSA Coding | 25 problems with complete solutions (Two Sum → Hit Counter) | [dsa-round.md](questions/dsa-round.md) |
| Low-Level Design | 25 problems with class sketches (Parking Lot → Config Manager) | [lld-problems.md](questions/lld-problems.md) |
| System Design (HLD) | 25 systems with architecture (URL Shortener → Multi-Region DB) | [hld-system-design.md](questions/hld-system-design.md) |
| CI/CD & DevOps | 25 problems with actual YAML/Dockerfile/Terraform/bash | [cicd-devops.md](questions/cicd-devops.md) |
| Architecture & Design | 25 trade-off discussions (Staff/Principal level) | [architecture-design.md](questions/architecture-design.md) |

**Total: 250+ questions with expected answers and code snippets.**
