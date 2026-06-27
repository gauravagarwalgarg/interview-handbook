# Quality Gates, Static Analysis & Agile Delivery

**Focus: SonarQube, clang-tidy, code coverage, test frameworks (GTest + pytest), software metrics, GitLab Agile workflow**

---

## ⚡ Quick-Fire (Quality & Process Gauge)

| Question | Expected Answer |
|----------|----------------|
| What is static analysis vs dynamic analysis? | Static: analyzes code without running (clang-tidy, Coverity). Dynamic: runs code (ASan, Valgrind, fuzzing). |
| What does SonarQube measure? | Code smells, bugs, vulnerabilities, coverage, duplication, complexity. Quality gates pass/fail. |
| What is clang-tidy? | Clang-based linter + modernizer. Checks coding style, performance, CERT/MISRA rules. Provides auto-fix. |
| What is LCOV/gcov? | Code coverage tools. gcov: per-file coverage from GCC. LCOV: HTML report generator on top of gcov. |
| What is cyclomatic complexity? | Number of independent paths through code. High = hard to test/maintain. Target: <10 per function. |
| What is a quality gate? | Pass/fail threshold on metrics (e.g., "no new critical bugs, coverage > 80%"). Blocks merge if failed. |
| GTest vs Catch2 vs doctest? | GTest: Google, full-featured, fixture support. Catch2: header-only, BDD style. doctest: fastest compile, minimal. |
| What is mutation testing? | Inserts bugs into code, runs tests  if tests still pass, they're weak. Measures test quality, not just coverage. |
| What is "shift left"? | Move quality checks earlier in development (IDE → commit → CI, not only before release). |
| What is technical debt? | Shortcuts/workarounds that increase future maintenance cost. Track in backlog, budget for payoff. |

---

## 🔬 Q1: GitLab CI Quality Pipeline

**Question:** Design a CI pipeline with quality gates for a C++ embedded project. It should block merges that:
- Introduce new clang-tidy warnings
- Drop code coverage below 75%
- Have SonarQube Quality Gate failure
- Fail any unit or integration test

**Expected answer:**

```yaml
# .gitlab-ci.yml  quality stages
stages:
  - lint
  - build
  - test
  - analyze
  - deploy

variables:
  CMAKE_BUILD_TYPE: Debug
  SONAR_HOST_URL: "https://sonar.company.com"

# --- Stage 1: Static Analysis ---
clang-tidy:
  stage: lint
  image: containers.global.bsf.tools/llvm-toolchain:17
  script:
    - cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
    - run-clang-tidy -p build/ -header-filter='src/.*' \
        -checks='-*,bugprone-*,performance-*,modernize-*,cert-*' \
        2>&1 | tee clang-tidy-report.txt
    - |
      # Fail if any warnings found in changed files
      ERRORS=$(grep "warning:" clang-tidy-report.txt | wc -l)
      if [ "$ERRORS" -gt 0 ]; then
        echo "❌ $ERRORS clang-tidy warnings found"
        exit 1
      fi
  artifacts:
    paths: [clang-tidy-report.txt]
    when: always

cppcheck:
  stage: lint
  image: containers.global.bsf.tools/cppcheck:2.13
  script:
    - cppcheck --enable=all --error-exitcode=1 \
        --suppress=missingIncludeSystem \
        --xml --xml-version=2 src/ 2> cppcheck-report.xml
  artifacts:
    paths: [cppcheck-report.xml]
    reports:
      codequality: cppcheck-report.xml

# --- Stage 2: Build ---
build:
  stage: build
  image: containers.global.bsf.tools/yocto-sdk:kirkstone
  script:
    - source /opt/poky/env-setup-cortexa53
    - cmake -B build -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTS=ON
        -DCMAKE_CXX_FLAGS="--coverage -fprofile-arcs -ftest-coverage"
    - cmake --build build -j$(nproc)
  artifacts:
    paths: [build/]

# --- Stage 3: Test + Coverage ---
unit-tests:
  stage: test
  image: containers.global.bsf.tools/yocto-sdk:kirkstone
  needs: [build]
  script:
    - cd build && ctest --output-on-failure --timeout 60
    - gcovr --xml-pretty --exclude='.*test.*' --print-summary
        -o coverage.xml --root ${CI_PROJECT_DIR}
    - gcovr --html-details coverage.html --root ${CI_PROJECT_DIR}
  coverage: '/^TOTAL.*\s+(\d+\%)$/'
  artifacts:
    paths: [build/coverage.html, build/coverage.xml]
    reports:
      coverage_report:
        coverage_format: cobertura
        path: build/coverage.xml

pytest-integration:
  stage: test
  image: containers.global.bsf.tools/python:3.12-slim
  needs: [build]
  script:
    - python3 -m pip install pytest pytest-html pytest-timeout
    - python3 -m pytest tests/integration/ --junitxml=pytest-report.xml
        --html=pytest-report.html --timeout=120
  artifacts:
    reports:
      junit: pytest-report.xml
    paths: [pytest-report.html]

# --- Stage 4: SonarQube Analysis ---
sonarqube:
  stage: analyze
  image: containers.global.bsf.tools/sonar-scanner:5
  needs: [unit-tests, clang-tidy, cppcheck]
  variables:
    SONAR_TOKEN: $SONAR_TOKEN
  script:
    - sonar-scanner
        -Dsonar.projectKey=${CI_PROJECT_PATH_SLUG}
        -Dsonar.sources=src/
        -Dsonar.tests=tests/
        -Dsonar.cfamily.compile-commands=build/compile_commands.json
        -Dsonar.coverageReportPaths=build/coverage.xml
        -Dsonar.qualitygate.wait=true
  allow_failure: false  # Blocks pipeline if quality gate fails
```


**Follow-up probes:**
- How do you handle false positives in clang-tidy? (`// NOLINT(check-name)` inline, or `.clang-tidy` config with exclude)
- How do you measure coverage on a cross-compiled binary? (Run on QEMU or target, copy `.gcda` back to build machine, run `gcovr` against `.gcno`)
- What SonarQube quality gate rules do you configure? (No new bugs, no new vulnerabilities, new code coverage > 80%, duplication < 3%)

---

## 🔬 Q2: GTest Framework for Embedded C++ (Unit Testing)

**Question:** Write a GTest test suite for a CAN message parser. Test:
- Valid frame parsing
- Invalid CRC rejection
- Buffer overflow protection
- Edge case: zero-length payload

**Expected answer:**

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include "can_parser.h"

class CanParserTest : public ::testing::Test {
protected:
    CanParser parser_;
    
    void SetUp() override {
        parser_.reset();
    }

    CanFrame make_valid_frame(uint16_t id, std::vector<uint8_t> payload) {
        CanFrame f{};
        f.id = id;
        f.dlc = payload.size();
        std::copy(payload.begin(), payload.end(), f.data);
        f.crc = compute_crc(f);
        return f;
    }
};

TEST_F(CanParserTest, ParsesValidFrame) {
    auto frame = make_valid_frame(0x100, {0x01, 0x02, 0x03, 0x04});
    
    auto result = parser_.parse(frame);
    
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->id, 0x100);
    EXPECT_EQ(result->payload_size, 4);
    EXPECT_EQ(result->payload[0], 0x01);
}

TEST_F(CanParserTest, RejectsInvalidCrc) {
    auto frame = make_valid_frame(0x200, {0xAA, 0xBB});
    frame.crc ^= 0xFF;  // Corrupt CRC
    
    auto result = parser_.parse(frame);
    
    ASSERT_FALSE(result.has_value());
    EXPECT_EQ(parser_.stats().crc_errors, 1);
}

TEST_F(CanParserTest, RejectsDlcOverflow) {
    CanFrame frame{};
    frame.id = 0x300;
    frame.dlc = 9;  // Invalid: max is 8 for classic CAN
    
    auto result = parser_.parse(frame);
    
    ASSERT_FALSE(result.has_value());
    EXPECT_EQ(parser_.stats().format_errors, 1);
}

TEST_F(CanParserTest, HandlesZeroLengthPayload) {
    auto frame = make_valid_frame(0x400, {});
    
    auto result = parser_.parse(frame);
    
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->payload_size, 0);
}

// Parameterized test for multiple valid IDs
class CanIdValidationTest : public CanParserTest,
                             public ::testing::WithParamInterface<uint16_t> {};

TEST_P(CanIdValidationTest, AcceptsValidId) {
    auto frame = make_valid_frame(GetParam(), {0x00});
    EXPECT_TRUE(parser_.parse(frame).has_value());
}

INSTANTIATE_TEST_SUITE_P(ValidIds, CanIdValidationTest,
    ::testing::Values(0x000, 0x001, 0x7FF));  // Standard CAN range

// Mock for hardware dependency
class MockCanBus : public ICanBus {
public:
    MOCK_METHOD(bool, send, (const CanFrame&), (override));
    MOCK_METHOD(std::optional<CanFrame>, receive, (std::chrono::milliseconds), (override));
};

TEST_F(CanParserTest, SendsAckOnValidFrame) {
    MockCanBus mock_bus;
    CanResponder responder(parser_, mock_bus);
    
    EXPECT_CALL(mock_bus, send(::testing::Field(&CanFrame::id, 0x7FF)))
        .Times(1);
    
    auto frame = make_valid_frame(0x100, {0x01});
    responder.handle(frame);
}
```

**Follow-up probes:**
- How do you run these on target hardware? (Cross-compile GTest, deploy to target, run via SSH from CI)
- What is test fixture lifetime? (SetUp/TearDown per test. SetUpTestSuite/TearDownTestSuite per suite.)
- How do you test time-dependent code? (Inject `IClock` interface, use fake clock in tests)

---

## 🔬 Q3: Software Metrics & Engineering Productivity

**Question:** As the platform engineer, you're asked to establish engineering metrics for your team. What do you track and why?

**Expected answer:**

| Metric | Tool | Target | Why |
|--------|------|--------|-----|
| Code coverage (line + branch) | gcovr / LCOV | >80% new code | Ensures test quality |
| Cyclomatic complexity | SonarQube / lizard | <10 per function | Maintainability |
| Build time | CI timestamps | <15 min feedback | Developer productivity |
| Mean time to merge (MR) | GitLab analytics | <24 hours | Flow efficiency |
| Bug escape rate | Jira/GitLab issues | Decreasing trend | Quality of testing |
| Static analysis findings | clang-tidy + SonarQube | Zero new on merge | Prevent debt accumulation |
| Binary size | `size` command in CI | Track regression | Constrained flash/RAM |
| Boot time | Automated boot test | <5 seconds | User experience |
| Dependency count | SBOM | Minimal, pinned | Supply chain security |
| Tech debt ratio | SonarQube | <5% | Long-term velocity |

**GitLab-native metrics:**
```yaml
# .gitlab-ci.yml metrics reporting
unit-tests:
  coverage: '/^TOTAL.*\s+(\d+\%)$/'  # Regex to extract coverage %
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
      junit: test-results.xml
```

**Binary size tracking script (CI):**
```python
#!/usr/bin/env python3
"""Track binary size regression in CI."""
import subprocess, json, sys
from pathlib import Path

def get_size(binary: Path) -> dict:
    result = subprocess.run(["size", str(binary)], capture_output=True, text=True)
    parts = result.stdout.splitlines()[1].split()
    return {"text": int(parts[0]), "data": int(parts[1]), "bss": int(parts[2]), "total": int(parts[3])}

current = get_size(Path(sys.argv[1]))
print(f"Binary size: text={current['text']} data={current['data']} bss={current['bss']} total={current['total']}")

# Compare against baseline (stored in CI artifact from main branch)
baseline_path = Path("metrics/binary_baseline.json")
if baseline_path.exists():
    baseline = json.loads(baseline_path.read_text())
    growth = current["total"] - baseline["total"]
    pct = (growth / baseline["total"]) * 100
    print(f"Size change: {growth:+d} bytes ({pct:+.1f}%)")
    if growth > 10240:  # >10KB growth = warning
        print("⚠️  Significant binary growth detected!")
        sys.exit(1)
```

---

## 🔬 Q4: Agile Delivery with GitLab

**Question:** How do you structure GitLab for a 6-person embedded team doing 2-week sprints?

**Expected answer:**

```
GitLab Project Structure:
├── Epics (large features spanning multiple sprints)
│   └── Epic: "OTA Update System v2"
├── Milestones (sprints / iterations)
│   ├── Sprint 24.1 (Jan 6-17)
│   └── Sprint 24.2 (Jan 20-31)
├── Issues (work items)
│   ├── Labels: priority::high, type::feature, type::bug, component::yocto
│   ├── Weight: story points (1,2,3,5,8)
│   └── Assignee + Due date
├── Merge Requests
│   ├── Linked to issues (closes #123)
│   ├── Require 1 reviewer approval
│   ├── Must pass CI pipeline
│   └── Auto-merge when approved + green
└── Boards
    ├── Dev Board: Open → In Progress → Review → Done
    └── Release Board: Staged → QA → Released
```

**Branching strategy:**

```
main ─────────────────────────────────────────────── (always deployable)
  │
  ├── feature/OTA-123-download-resume ──── MR → main
  ├── feature/SENS-45-new-imu-driver ──── MR → main
  └── release/v2.5 ─── cherry-picks → tag → deploy
```

**Key practices:**
- **Trunk-based development:** Short-lived feature branches (<3 days), merge to main frequently
- **MR requirements:** Pipeline green + 1 approval + no threads unresolved
- **Release:** Branch from main, only cherry-pick fixes, tag + build final artifacts
- **Retrospectives:** Track velocity, blocked time, escaped bugs per sprint

---

## 🔬 Q5: SonarQube Integration for C++ Projects

**Question:** Configure SonarQube for a C++ embedded project. What specific rules/profiles do you enable for safety-critical code?

**Expected answer:**

```properties
# sonar-project.properties
sonar.projectKey=company_sensor-platform
sonar.projectName=Sensor Platform
sonar.projectVersion=2.5.0

sonar.sources=src/
sonar.tests=tests/
sonar.sourceEncoding=UTF-8

# C/C++ specific
sonar.cfamily.compile-commands=build/compile_commands.json
sonar.cfamily.threads=4

# Coverage
sonar.coverageReportPaths=build/coverage.xml

# Quality Gate: embedded-critical profile
sonar.qualitygate.wait=true

# Exclusions
sonar.exclusions=**/generated/**,**/third_party/**
sonar.coverage.exclusions=**/test_*.cpp,**/mock_*.cpp
```

**Custom quality profile rules for embedded:**
- All CERT C++ rules (SEI CERT Coding Standard)
- Buffer overflow detections (CWE-120, CWE-131)
- Integer overflow checks
- Null pointer dereference
- Resource leak detection
- Thread-safety violations
- Dead code / unreachable code
- Unused variables / parameters

**Quality Gate configuration:**
```
Conditions for new code:
  ├── Coverage on new code ≥ 80%
  ├── Duplications on new code < 3%
  ├── Maintainability rating = A
  ├── Reliability rating = A (no new bugs)
  ├── Security rating = A (no new vulnerabilities)
  └── Security hotspots reviewed = 100%
```

---

## ⚡ Quality & Process Quick-Fire

| Question | Expected Answer |
|----------|----------------|
| What is DORA metrics? | Deployment Frequency, Lead Time for Changes, Change Failure Rate, MTTR. Measures DevOps performance. |
| What is a "definition of done"? | Checklist: code reviewed, tests pass, coverage met, docs updated, QA verified, deployed to staging. |
| How do you handle flaky tests? | Quarantine (separate job), retry with limit, investigate root cause. Never permanently ignore. |
| What is trunk-based development? | All developers commit to main (via short-lived branches). No long-lived feature branches. CI always green. |
| What is a merge train? | Sequential MR testing: each MR tested with all prior queued changes merged. Prevents broken main. |
| What is the boy scout rule? | Leave code cleaner than you found it. Small refactors with each change. Prevents debt accumulation. |
| How do you prioritize tech debt? | Impact × Likelihood framework. Fix debt that blocks features or causes bugs first. Budget 20% sprint capacity. |
| What is an ADR? | Architecture Decision Record. Captures context, decision, consequences. Version-controlled. Prevents repeated debates. |
