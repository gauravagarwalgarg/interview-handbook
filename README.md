# Interview Question Bank

**Embedded Systems | Linux Kernel | C++ | Yocto | Python**

A comprehensive, production-oriented interview question bank for roles requiring a blend of C++, Embedded Systems, Linux, Yocto Project, and Python scripting. Designed for 60-minute technical rounds with a focus on real-world engineering challenges.

---

## Structure

| Section | Focus Area |
|---------|-----------|
| [C++ (Modern C++14/17/20)](C++/Overview.md) | Memory management, RAII, concurrency, design patterns, OOP for constrained environments |
| [C++ Tricky Output Questions](C++/TrickyOutputQuestions.md) | Static variables, virtual dispatch, object lifetime, UB  "what does this print?" |
| [C++ Architecture (Senior/Staff)](C++/Architecture.md) | Layered platform design, API contracts, DI, plugin systems |
| [Build Systems (CMake/Ninja)](C++/BuildSystems_CMake.md) | CMakeLists.txt, cross-compilation, target-based design |
| [Embedded Systems & Linux](EmbeddedSystems/Overview.md) | MCU vs MPU, interrupts, RTOS, Linux boot, drivers, IPC |
| [Protocols & Networking](EmbeddedSystems/Protocols_Networking.md) | CAN, UART, SPI, I2C, REST/HTTP, MQTT, Wi-Fi, BLE |
| [Yocto Project](Yocto/Overview.md) | Recipes, layers, bitbake, image customization, debugging |
| [Yocto for Zynq/ZynqMP](Yocto/Zynq.md) | meta-xilinx, FPGA integration, PS-PL, boot flow, UIO |
| [CI/CD & Cloud](CICD_Cloud/Overview.md) | Build pipelines, fleet management, web dashboards, Docker |
| [Quality Gates & Agile](CICD_Cloud/QualityTooling.md) | SonarQube, clang-tidy, GTest, coverage, GitLab Agile |
| [Python Scripting](Python/Overview.md) | Automation, log parsing, serial comms, pytest |
| [Cross-Functional Scenarios](CrossFunctional/Overview.md) | Integration scenarios across all domains |
| [Candidate Evaluation Metrics](Metrics/EvaluationSheet.md) | Scoring rubric and feedback template |

---

## Interview Flow (Recommended 60-min Structure)

### Round 1: Platform & C++ Deep Dive (Senior / Staff, 9+ Years)

| Time | Phase | JD Area Covered |
|------|-------|-----------------|
| 0–5 min | Quick-fire + Tricky Output Qs | C/C++ language depth gauge |
| 5–20 min | C++ Architecture + CMake | Modern C++, build systems, API design |
| 20–35 min | Protocols + Embedded Linux | CAN/SPI/I2C, REST/MQTT, IPC, Linux drivers |
| 35–50 min | Yocto Platform | Layers, recipes, image customization, SDK |
| 50–58 min | Design Pattern Exercise | State machine / Observer / Factory (whiteboard) |
| 58–60 min | Candidate Questions | |

### Round 2: Delivery, Quality & Systems Integration

| Time | Phase | JD Area Covered |
|------|-------|-----------------|
| 0–5 min | Quick-fire (CI/CD + Quality) | Pipeline fluency, SonarQube, coverage |
| 5–20 min | GitLab CI/CD Pipeline Design | Quality gates, containerized builds, release automation |
| 20–30 min | Testing Strategy | GTest, pytest, HW-in-loop, coverage |
| 30–45 min | Cross-Functional Scenario | End-to-end: Yocto build → CI → OTA deploy |
| 45–55 min | Agile & Software Metrics | Sprint execution, tracking, engineering productivity |
| 55–60 min | Candidate Questions | |

### For Mid-Level (4-8 Years)

| Time | Phase | Purpose |
|------|-------|---------|
| 0–5 min | Warm-up / Quick-fire | Gauge language fluency (brief yes/no/why questions) |
| 5–20 min | C++ Deep Dive | Design pattern exercise + memory/concurrency probing |
| 20–35 min | Embedded + Linux | Platform-level understanding, driver/boot knowledge |
| 35–45 min | Yocto / Build System | Practical build troubleshooting |
| 45–55 min | Cross-Functional Scenario | End-to-end problem solving |
| 55–60 min | Candidate Questions | |

---

## Usage Notes

- Questions marked with `⚡` are quick-fire (30s expected answer).
- Questions marked with `🔬` are deep-dive (5–10 min discussion).
- Questions marked with `🏗️` are design/whiteboard exercises.
- Each section contains an **Answer Key / Evaluation Criteria** for interviewers.
- The [Metrics Sheet](Metrics/EvaluationSheet.md) should be filled during or immediately after the interview.

---

## 📖 View as Documentation Site

This repository is configured with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) for a polished browsing experience.

```bash
# Install dependencies
pip install -r requirements-docs.txt

# Serve locally with hot-reload
mkdocs serve
# → Open http://127.0.0.1:8000

# Build static site
mkdocs build

# Deploy to GitHub Pages
mkdocs gh-deploy
```

**CI/CD:** On push to `main`, the site auto-deploys via GitHub Actions (`.github/workflows/mkdocs.yml`) or GitLab Pages (`.gitlab-ci.yml`).
