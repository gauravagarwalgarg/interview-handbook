# CI/CD & Cloud Integration  Platform Engineering

**Focus: Build pipelines, automated testing, device fleet management, web dashboards, containerized builds**

---

## ⚡ Quick-Fire (CI/CD & Cloud Gauge)

| Question | Expected Answer |
|----------|----------------|
| What is the difference between CI and CD? | CI: automated build + test on every commit. CD: automated deployment to staging/production. |
| GitLab CI vs Jenkins for embedded? | GitLab: YAML-native, integrated MR workflow. Jenkins: more flexible agents, legacy Yocto support. |
| What is a Docker multi-stage build? | Build in heavy image (compilers, Yocto), copy artifacts to slim runtime image. Reduces final size. |
| What is an artifact in CI? | Build output (image, binary, report) stored for downstream jobs or deployment. |
| What is infrastructure as code? | Managing infra via version-controlled files (Terraform, CloudFormation) instead of manual setup. |
| What is MQTT vs HTTP for IoT? | MQTT: lightweight pub/sub, persistent sessions, QoS levels, low bandwidth. HTTP: request/response, heavier. |
| What is a container registry? | Storage for Docker/OCI images (ECR, GitLab Registry, Harbor). Versioned, pullable by CI/devices. |
| What is `sstate` caching in CI? | Storing Yocto shared-state artifacts (S3, NFS) to avoid rebuilding unchanged recipes. 80%+ build time savings. |

---

## 🔬 Q1: End-to-End CI/CD Pipeline for Zynq Platform

**Question:** Design the complete CI/CD pipeline for a Zynq-based product
that goes from developer commit to deployed firmware on 200 field devices.

**Expected pipeline design:**

```yaml
# .gitlab-ci.yml (conceptual)
stages:
  - validate
  - build-fpga
  - build-linux
  - test-hw
  - release
  - deploy
```


```
┌──────────────────────────────────────────────────────────────────────┐
│                        CI/CD Pipeline                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌─────────────────┐ │
│  │Validate │───▶│Build FPGA│───▶│Build Linux│───▶│ HW-in-Loop Test │ │
│  │(lint,   │    │(Vivado   │    │(Yocto    │    │ (QEMU + real    │ │
│  │ static  │    │ synthesis│    │ bitbake) │    │  board farm)    │ │
│  │analysis)│    │ ~2-4 hrs)│    │ ~30 min  │    │                 │ │
│  └─────────┘    └──────────┘    └──────────┘    └────────┬────────┘ │
│                                                           │          │
│                                                           ▼          │
│  ┌─────────────────┐    ┌────────────────────────────────────────┐  │
│  │ Release & Sign  │◀───│ Manual Approval Gate                    │  │
│  │ (bootgen, gpg)  │    │ (tech lead reviews test report)        │  │
│  └────────┬────────┘    └────────────────────────────────────────┘  │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ Deploy (OTA via Mender/RAUC → AWS IoT / Azure IoT Hub)          ││
│  │  - Staged rollout: 5% → 25% → 100%                              ││
│  │  - Automatic rollback on health check failure                    ││
│  └─────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────┘
```

**Detailed stage breakdown:**

```yaml
validate:
  stage: validate
  image: company/cpp-lint:latest
  script:
    - clang-tidy --config-file=.clang-tidy src/**/*.cpp
    - cppcheck --enable=all --error-exitcode=1 src/
    - python3 -m pylint scripts/ --fail-under=8.0
    - yamllint .gitlab-ci.yml mkdocs.yml
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

build-fpga:
  stage: build-fpga
  tags: [vivado-runner]  # Dedicated machine with Vivado license
  script:
    - source /opt/Xilinx/Vivado/2023.2/settings64.sh
    - make -C fpga/ bitstream
    - bootgen -image fpga/boot.bif -o BOOT.BIN -w
  artifacts:
    paths: [fpga/output/*.bit.bin, fpga/output/*.xsa, BOOT.BIN]
  cache:
    key: vivado-$CI_COMMIT_REF_SLUG
    paths: [fpga/.Xil/]
  rules:
    - changes: [fpga/**/*]  # Only rebuild if FPGA sources changed

build-linux:
  stage: build-linux
  image: crops/poky:ubuntu-22.04
  variables:
    SSTATE_MIRROR: "http://sstate-cache.company.local/sstate-cache"
    DL_MIRROR: "http://dl-mirror.company.local/downloads"
  script:
    - kas build kas/production.yml
  artifacts:
    paths:
      - build/tmp/deploy/images/custom-zynqmp/*.wic.gz
      - build/tmp/deploy/images/custom-zynqmp/BOOT.BIN
      - build/tmp/deploy/sdk/*.sh
  cache:
    key: yocto-sstate
    paths: [build/sstate-cache/]

test-hw:
  stage: test-hw
  tags: [board-farm]  # Runner with physical Zynq board attached
  needs: [build-linux, build-fpga]
  script:
    - python3 -m pytest tests/hw/ --junitxml=report.xml --timeout=600
    - python3 scripts/flash_and_boot.py --image $ARTIFACT_PATH
    - python3 -m pytest tests/integration/ --target=192.168.1.100
  artifacts:
    reports:
      junit: report.xml
```

**Follow-up probes:**
- How do you handle Vivado licenses in CI? (Floating license server, or run on dedicated machine with node-locked license)
- How do you speed up 4-hour FPGA builds? (Incremental synthesis, OOC for submodules, run only on FPGA source changes)
- How do you handle flaky HW tests? (Retry with exponential backoff, quarantine flaky tests, separate reliability suite)

---

## 🔬 Q2: Containerized Yocto Builds

**Question:** Design the Docker setup for reproducible Yocto builds in CI.

**Expected answer:**

```dockerfile
# Dockerfile.yocto-builder
FROM ubuntu:22.04

# Yocto host dependencies
RUN apt-get update && apt-get install -y \
    gawk wget git diffstat unzip texinfo gcc \
    build-essential chrpath socat cpio python3 \
    python3-pip python3-pexpect xz-utils debianutils \
    iputils-ping python3-git python3-jinja2 libegl1-mesa \
    libsdl1.2-dev pylint xterm python3-subunit locales \
    mesa-common-dev zstd liblz4-tool file \
    && rm -rf /var/lib/apt/lists/*

# Locale (Yocto requirement)
RUN locale-gen en_US.UTF-8
ENV LANG=en_US.UTF-8

# Non-root user (Yocto refuses root builds)
RUN useradd -m -s /bin/bash builder
USER builder
WORKDIR /home/builder

# Install kas (Yocto build orchestrator)
RUN pip3 install --user kas

ENV PATH="/home/builder/.local/bin:${PATH}"
```

```yaml
# docker-compose.yml for local development
services:
  yocto-build:
    build:
      context: .
      dockerfile: Dockerfile.yocto-builder
    volumes:
      - ./:/home/builder/project
      - yocto-sstate:/home/builder/sstate-cache
      - yocto-downloads:/home/builder/downloads
    environment:
      - SSTATE_DIR=/home/builder/sstate-cache
      - DL_DIR=/home/builder/downloads
    command: kas build kas/development.yml

volumes:
  yocto-sstate:
  yocto-downloads:
```

**Key considerations:**
- Named volumes for sstate + downloads (persist across container restarts)
- Non-root user inside container (Yocto requirement)
- CI runner mounts NFS sstate cache for team-wide sharing
- `kas` handles layer checkout + build configuration in one command

---

## 🔬 Q3: Device Fleet Management & Cloud Architecture

**Question:** Design the cloud backend for managing 200+ Zynq devices in the field.

**Expected architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                        Cloud (AWS / Azure)                        │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐│
│  │ IoT Core /   │  │ S3 / Blob    │  │ Web Dashboard          ││
│  │ IoT Hub      │  │ (firmware    │  │ (React + REST API)     ││
│  │ (MQTT broker)│  │  artifacts)  │  │                        ││
│  └──────┬───────┘  └──────┬───────┘  └────────────┬───────────┘│
│         │                  │                       │             │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌────────────▼───────────┐│
│  │ Lambda /     │  │ CloudFront / │  │ API Gateway +          ││
│  │ Functions    │  │ CDN          │  │ Flask/FastAPI Backend   ││
│  │ (telemetry   │  │ (firmware    │  │                        ││
│  │  processing) │  │  delivery)   │  │                        ││
│  └──────┬───────┘  └──────────────┘  └────────────────────────┘│
│         │                                                        │
│  ┌──────▼───────┐  ┌──────────────┐  ┌────────────────────────┐│
│  │ TimescaleDB /│  │ DynamoDB /   │  │ Grafana / CloudWatch   ││
│  │ InfluxDB     │  │ CosmosDB     │  │ (monitoring)           ││
│  │ (time-series)│  │ (device state│  │                        ││
│  └──────────────┘  └──────────────┘  └────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                              ▲ MQTT/TLS
                              │
                    ┌─────────┴─────────┐
                    │   Field Devices    │
                    │   (Zynq + Linux)   │
                    │                    │
                    │  ┌──────────────┐  │
                    │  │ MQTT Client  │  │
                    │  │ (C++ daemon) │  │
                    │  └──────────────┘  │
                    └───────────────────-┘
```


**Device-side MQTT telemetry (C++):**

```cpp
class CloudConnector {
public:
    struct Config {
        std::string broker_url;     // "mqtts://iot.company.com:8883"
        std::string device_id;
        std::filesystem::path cert_path;
        std::filesystem::path key_path;
        std::filesystem::path ca_path;
        std::chrono::seconds telemetry_interval{60};
        std::chrono::seconds heartbeat_interval{30};
    };

    explicit CloudConnector(Config config) : config_(std::move(config)) {}

    bool connect() {
        // TLS mutual authentication (X.509 device certificate)
        client_.set_tls(config_.ca_path, config_.cert_path, config_.key_path);
        client_.set_will(status_topic(), R"({"status":"offline"})", 1, true);
        return client_.connect(config_.broker_url, config_.device_id);
    }

    void publish_telemetry(const TelemetryData& data) {
        auto json = serialize_to_json(data);
        client_.publish(telemetry_topic(), json, /*qos=*/1);
    }

    void subscribe_commands() {
        client_.subscribe(command_topic(), /*qos=*/1, [this](auto& msg) {
            handle_command(msg.payload());
        });
    }

private:
    std::string telemetry_topic() const {
        return fmt::format("dt/{}/telemetry", config_.device_id);
    }
    std::string command_topic() const {
        return fmt::format("cmd/{}/+", config_.device_id);
    }
    std::string status_topic() const {
        return fmt::format("dt/{}/status", config_.device_id);
    }

    void handle_command(std::string_view payload) {
        // Parse JSON command, dispatch to appropriate handler
        // e.g., "reboot", "update_config", "start_diagnostics"
    }

    Config config_;
    MqttClient client_;
};
```

**Follow-up probes:**
- How do you provision device certificates at factory? (Secure element + CSR generated on-device, signed by PKI during manufacturing)
- What MQTT QoS do you use for telemetry vs commands? (Telemetry: QoS 1, can tolerate occasional loss. Commands: QoS 1 with ACK.)
- How do you handle offline periods? (Local buffer to flash, replay on reconnection with timestamps)
- Shadow/twin concept? (AWS IoT Device Shadow / Azure Device Twin  desired vs reported state)

---

## 🔬 Q4: Web Dashboard for Device Monitoring (Python/FastAPI)

**Question:** Design the backend API for a web dashboard that shows:
- Real-time device status (online/offline, last heartbeat)
- Telemetry charts (temperature, motor position over time)
- Firmware version across fleet
- OTA update management (trigger, monitor progress)

**Expected answer (FastAPI):**

```python
from fastapi import FastAPI, WebSocket, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from datetime import datetime
from typing import Optional
import asyncio

app = FastAPI(title="Device Fleet API", version="2.0.0")

app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"])

# --- Models ---
class DeviceStatus(BaseModel):
    device_id: str
    online: bool
    firmware_version: str
    last_heartbeat: datetime
    uptime_hours: float
    ip_address: Optional[str]

class TelemetryPoint(BaseModel):
    timestamp: datetime
    temperature: float
    motor_position: float
    cpu_load: float

class OtaRequest(BaseModel):
    device_ids: list[str]
    target_version: str
    rollout_percentage: int = 100
    schedule: Optional[datetime] = None

# --- REST Endpoints ---
@app.get("/api/v1/devices", response_model=list[DeviceStatus])
async def list_devices(online_only: bool = False):
    """List all devices with current status."""
    devices = await db.get_all_devices()
    if online_only:
        devices = [d for d in devices if d.online]
    return devices

@app.get("/api/v1/devices/{device_id}/telemetry")
async def get_telemetry(device_id: str, hours: int = 24, resolution: str = "1m"):
    """Get time-series telemetry for a device."""
    return await timeseries_db.query(
        device_id=device_id,
        duration_hours=hours,
        downsample=resolution
    )

@app.post("/api/v1/ota/deploy")
async def trigger_ota(request: OtaRequest):
    """Trigger OTA update to selected devices."""
    # Validate firmware version exists in artifact store
    if not await artifact_store.version_exists(request.target_version):
        raise HTTPException(404, "Firmware version not found")

    job = await ota_service.create_deployment(
        device_ids=request.device_ids,
        version=request.target_version,
        rollout_pct=request.rollout_percentage,
    )
    return {"job_id": job.id, "status": "scheduled"}

@app.get("/api/v1/ota/jobs/{job_id}")
async def get_ota_status(job_id: str):
    """Get OTA deployment progress."""
    return await ota_service.get_job_status(job_id)

# --- WebSocket for real-time updates ---
@app.websocket("/ws/devices/{device_id}/live")
async def device_live_stream(websocket: WebSocket, device_id: str):
    """Stream real-time telemetry to dashboard."""
    await websocket.accept()
    try:
        async for data in telemetry_stream.subscribe(device_id):
            await websocket.send_json(data)
    except Exception:
        await websocket.close()

# --- Fleet overview ---
@app.get("/api/v1/fleet/summary")
async def fleet_summary():
    """Aggregate fleet health metrics."""
    devices = await db.get_all_devices()
    return {
        "total": len(devices),
        "online": sum(1 for d in devices if d.online),
        "firmware_versions": Counter(d.firmware_version for d in devices),
        "avg_uptime_hours": mean(d.uptime_hours for d in devices),
        "devices_needing_update": sum(
            1 for d in devices if d.firmware_version != LATEST_VERSION
        ),
    }
```

---

## 🔬 Q5: Testing Strategy Across the Stack

**Question:** For a 9+ year platform engineer, describe how you ensure
quality across: FPGA, firmware, Linux image, cloud backend, web UI.

**Expected answer  Test pyramid for embedded:**

```
                    ┌─────────────────┐
                    │ System Tests    │  ← Full stack: HW + Cloud + UI
                    │ (few, expensive)│     Weekly, on board farm
                    ├─────────────────┤
                    │ Integration     │  ← QEMU + mock cloud
                    │ Tests           │     Per merge request
                    ├─────────────────┤
                    │ Component Tests │  ← Individual services
                    │                 │     Docker compose
                    ├─────────────────┤
                    │ Unit Tests      │  ← C++ (gtest), Python (pytest)
                    │ (many, fast)    │     Every commit, <5 min
                    └─────────────────┘
```

| Layer | Tool | What It Tests | Run When |
|-------|------|---------------|----------|
| C++ unit | Google Test + GMock | Business logic, state machines, algorithms | Every push |
| C++ static | clang-tidy, cppcheck, MISRA checker | Code quality, potential bugs | Every push |
| C++ sanitizers | ASan, TSan, UBSan | Memory errors, races, UB | Nightly |
| FPGA simulation | Vivado xsim, Verilator | RTL correctness | FPGA changes |
| Linux boot | QEMU + pytest | Image boots, services start | Every build |
| Integration | Docker compose + device simulator | API endpoints, MQTT flow | Per MR |
| HW-in-loop | Board farm + pytest | Real hardware behavior | Nightly / release |
| Cloud API | pytest + httpx | REST endpoints, WebSocket | Every push |
| Web UI | Cypress / Playwright | User workflows | Per MR |
| Performance | Custom benchmarks | Latency, throughput regression | Weekly |
| Security | Trivy (containers), Snyk (deps) | CVEs, dependency vulnerabilities | Daily |

---

## 🔬 Q6: Infrastructure as Code for Build Farm

**Question:** Design the infrastructure for a Yocto build farm that supports
5 developers + CI with fast feedback.

**Expected answer (Terraform + AWS):**

```hcl
# Yocto build server (high-performance)
resource "aws_instance" "yocto_builder" {
  ami           = "ami-ubuntu-22.04"
  instance_type = "c6i.16xlarge"  # 64 vCPU, 128 GB RAM
  
  root_block_device {
    volume_type = "gp3"
    volume_size = 500  # Yocto needs lots of disk
    iops        = 10000
  }

  tags = { Name = "yocto-builder-01" }
}

# Shared sstate cache on EFS
resource "aws_efs_file_system" "sstate" {
  performance_mode = "generalPurpose"
  throughput_mode  = "bursting"
  
  tags = { Name = "yocto-sstate-cache" }
}

# Downloads mirror on S3
resource "aws_s3_bucket" "downloads" {
  bucket = "company-yocto-downloads-mirror"
}

# GitLab runner registration
resource "aws_instance" "gitlab_runner" {
  ami           = "ami-ubuntu-22.04"
  instance_type = "c6i.8xlarge"
  
  user_data = <<-EOF
    #!/bin/bash
    curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | bash
    apt-get install gitlab-runner
    gitlab-runner register --url https://gitlab.company.com --token $TOKEN
  EOF
}
```

**Key infrastructure decisions:**
- EFS for sstate cache (shared across all builders, NFS-compatible)
- S3 for downloads mirror (persistent, cheap, `DL_DIR` populates it)
- Spot instances for CI builders (70% cost reduction, acceptable for non-prod)
- Dedicated builder for FPGA (needs Vivado license + lots of RAM)

---

## ⚡ CI/CD Quick-Fire (Senior Gauge)

| Question | Expected Answer |
|----------|----------------|
| What is GitOps? | Infrastructure/deployment defined in Git. Changes via MR. Automated reconciliation. |
| Blue-green vs canary deployment? | Blue-green: swap entire fleet at once. Canary: roll out to subset first, monitor, expand. |
| How do you handle secrets in CI? | CI variables (masked), Vault integration, or cloud secret managers. Never in repo. |
| What is a merge train? | Queued MRs that are tested in sequence with each other's changes merged. Prevents broken main. |
| What is SBOM and why generate it? | Software Bill of Materials. Required for supply-chain security, compliance, vulnerability tracking. |
| Container vs VM for CI runners? | Container: fast startup, lightweight, good for most builds. VM: needed for Yocto (large disk, privileged ops). |
| How do you handle a broken main branch? | Auto-revert the offending commit, notify author, require fix-forward or revert MR. |
| What is shift-left testing? | Move testing earlier in pipeline. Static analysis before build, unit tests before integration. |
