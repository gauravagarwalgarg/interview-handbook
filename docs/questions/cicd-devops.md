# CI/CD & DevOps Questions

> 25 real infra/DevOps questions with actual scripts, manifests, and architecture decisions.
> Focus: Docker, Kubernetes, Terraform, GitHub Actions, observability, and deployment strategies.

---

### Q1: Multi-Stage Dockerfile for Go Microservice Level: SDE 2

**Question:** Write a production-ready Dockerfile for a Go service.

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

# Runtime stage
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
COPY --from=builder /app/configs /configs
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Key decisions:** Multi-stage reduces image from ~800MB to ~10MB. Distroless has no shell (reduces attack surface). `CGO_ENABLED=0` for static binary. `-ldflags="-s -w"` strips debug info.

**Follow-up:** How to cache Go modules layer? → `COPY go.mod go.sum` before source code. Layer cache invalidates only when deps change.

---

### Q2: GitHub Actions Workflow Test + Deploy Level: SDE 2

**Question:** Write a CI/CD pipeline that tests, builds, and deploys.

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - name: Cache Go modules
        uses: actions/cache@v4
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('go.sum') }}
      - run: go test ./... -race -coverprofile=coverage.out
      - run: go vet ./...

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: |
          kubectl set image deployment/app \
            app=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --namespace=production
```

**Follow-up:** How to add manual approval gate? → Use `environment: production` with protection rules.

---

### Q3: GitLab CI Pipeline with Caching Level: SDE 2

**Question:** Write a GitLab CI with parallel testing, caching, and deployment.

```yaml
stages:
  - test
  - build
  - deploy

variables:
  GOPATH: $CI_PROJECT_DIR/.go

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .go/pkg/mod/

test:unit:
  stage: test
  image: golang:1.22
  script:
    - go test ./... -race -count=1
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

test:lint:
  stage: test
  image: golangci/golangci-lint:latest
  script:
    - golangci-lint run --timeout 5m

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy:production:
  stage: deploy
  image: bitnami/kubectl
  script:
    - kubectl set image deployment/app app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  environment:
    name: production
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

### Q4: Blue-Green Deployment Strategy Level: SDE 3

**Question:** Design a blue-green deployment with instant rollback.

**Architecture:**
```
Load Balancer
   ├── Blue (current production, v1.0)
   └── Green (new version, v1.1, pre-warmed)

Deployment steps:
1. Deploy v1.1 to Green environment
2. Run smoke tests against Green
3. Switch LB traffic from Blue → Green (instant)
4. Monitor for 15 min
5. If OK: decommission Blue (or keep as rollback)
6. If bad: switch LB back to Blue (instant rollback)
```

**Implementation (AWS):**
```bash
#!/bin/bash
# Switch ALB target group
BLUE_TG="arn:aws:elasticloadbalancing:...:targetgroup/blue/..."
GREEN_TG="arn:aws:elasticloadbalancing:...:targetgroup/green/..."
LISTENER_ARN="arn:aws:elasticloadbalancing:...:listener/..."

# Point listener to green
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$GREEN_TG

echo "Traffic switched to Green. Monitor and confirm."
```

**Trade-offs:** Requires 2× infrastructure during deployment. Database migrations must be backward-compatible.

---

### Q5: Kubernetes Deployment + HPA Level: SDE 2

**Question:** Write a K8s deployment with HPA and health checks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: ghcr.io/company/api:v1.2.3
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

**Follow-up:** Difference between liveness and readiness probes? → Liveness: restart if unhealthy. Readiness: remove from service endpoints (no traffic).

---

### Q6: Health Check Script Level: SDE 1

**Question:** Write a bash health check for a service.

```bash
#!/bin/bash
set -euo pipefail

SERVICE_URL="${1:-http://localhost:8080/healthz}"
TIMEOUT=5
RETRIES=3
DELAY=2

check_health() {
    local status
    status=$(curl -s -o /dev/null -w "%{http_code}" --max-time $TIMEOUT "$SERVICE_URL")
    [[ "$status" == "200" ]]
}

for i in $(seq 1 $RETRIES); do
    if check_health; then
        echo "✓ Service healthy (attempt $i)"
        exit 0
    fi
    echo "✗ Health check failed (attempt $i/$RETRIES)"
    [[ $i -lt $RETRIES ]] && sleep $DELAY
done

echo "CRITICAL: Service unhealthy after $RETRIES attempts"
exit 1
```

---

### Q7: Canary Release with Rollback Level: SDE 3

**Question:** Design canary release with automatic rollback.

```yaml
# Argo Rollouts canary strategy
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-server
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5    # 5% traffic to canary
        - pause:
            duration: 5m
        - analysis:
            templates:
              - templateName: error-rate-check
        - setWeight: 25
        - pause:
            duration: 10m
        - analysis:
            templates:
              - templateName: latency-check
        - setWeight: 75
        - pause:
            duration: 10m
        - setWeight: 100
      analysis:
        successfulRunHistoryLimit: 3
        unsuccessfulRunHistoryLimit: 1
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{status=~"5..",app="api-server",version="canary"}[5m]))
            / sum(rate(http_requests_total{app="api-server",version="canary"}[5m]))
      successCondition: result[0] < 0.01  # <1% error rate
      failureLimit: 2
```

---

### Q8: Terraform for AWS VPC + ECS Level: SDE 3

**Question:** Write Terraform for a VPC with ECS Fargate service.

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "app-vpc" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "private-${count.index}" }
}

resource "aws_ecs_cluster" "main" {
  name = "app-cluster"
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_service" "app" {
  name            = "app-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.app.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8080
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = "app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"

  container_definitions = jsonencode([{
    name  = "app"
    image = "123456789.dkr.ecr.us-east-1.amazonaws.com/app:latest"
    portMappings = [{ containerPort = 8080 }]
    logConfiguration = {
      logDriver = "awslogs"
      options   = { awslogs-group = "/ecs/app", awslogs-region = "us-east-1" }
    }
  }])
}
```

---

### Q9: Monorepo CI/CD Pipeline Design Level: SDE 3

**Question:** Design CI/CD for a monorepo with 5 services.

**Architecture:**
```
monorepo/
├── services/
│   ├── auth/
│   ├── billing/
│   ├── api/
│   ├── worker/
│   └── notification/
├── libs/
│   └── shared/
└── .github/workflows/
```

**Strategy:**
```yaml
# Detect changed services
- name: Detect changes
  id: changes
  uses: dorny/paths-filter@v3
  with:
    filters: |
      auth: ['services/auth/**', 'libs/shared/**']
      billing: ['services/billing/**', 'libs/shared/**']
      api: ['services/api/**', 'libs/shared/**']

# Build only affected services
- name: Build auth
  if: steps.changes.outputs.auth == 'true'
  run: make build-auth
```

**Key decisions:** Path-based triggers (only build what changed). Shared lib change triggers all dependents. Parallel builds per service. Shared artifact cache.

---

### Q10: Log Rotation Script Level: SDE 1

**Question:** Write a log rotation script.

```bash
#!/bin/bash
LOG_DIR="${1:-/var/log/app}"
MAX_FILES=7
MAX_SIZE_MB=100

rotate_log() {
    local file="$1"
    local size_mb
    size_mb=$(du -m "$file" | awk '{print $1}')
    
    if [[ $size_mb -ge $MAX_SIZE_MB ]]; then
        local timestamp
        timestamp=$(date +%Y%m%d_%H%M%S)
        mv "$file" "${file}.${timestamp}"
        gzip "${file}.${timestamp}" &
        touch "$file"
        echo "Rotated: $file (${size_mb}MB)"
    fi
}

cleanup_old() {
    local base_name="$1"
    ls -t "${base_name}".* 2>/dev/null | tail -n +$((MAX_FILES + 1)) | xargs -r rm -f
}

for log_file in "$LOG_DIR"/*.log; do
    [[ -f "$log_file" ]] || continue
    rotate_log "$log_file"
    cleanup_old "$log_file"
done
```

---

### Q11: Secrets Management with Vault Level: SDE 3

**Question:** Design secrets management integration.

**Architecture:**
```
App Startup → Vault Agent (sidecar) → HashiCorp Vault → Secrets Engine
            → Inject into env/files    ↑ (AppRole auth)
            → Auto-rotation             → Audit log
```

**K8s Integration:**
```yaml
# Vault Agent Injector annotations
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-inject-secret-db: "database/creds/app-role"
  vault.hashicorp.com/agent-inject-template-db: |
    {{- with secret "database/creds/app-role" -}}
    DB_USER={{ .Data.username }}
    DB_PASS={{ .Data.password }}
    {{- end }}
  vault.hashicorp.com/role: "app"
```

**Key decisions:** Dynamic secrets (generated on-demand, auto-expire). Lease-based rotation. Audit all access. Encryption as a service for application data.

---

### Q12: Database Migration Script Level: SDE 2

**Question:** Write a safe database migration with rollback.

```bash
#!/bin/bash
set -euo pipefail

DB_URL="${DATABASE_URL}"
MIGRATIONS_DIR="./migrations"
LOCK_TABLE="migration_lock"

acquire_lock() {
    psql "$DB_URL" -c "
        INSERT INTO $LOCK_TABLE (locked, locked_by, locked_at)
        VALUES (true, '$(hostname)', NOW())
        ON CONFLICT (id) DO UPDATE SET locked=true
        WHERE NOT migration_lock.locked;" || { echo "Migration locked"; exit 1; }
}

release_lock() {
    psql "$DB_URL" -c "UPDATE $LOCK_TABLE SET locked=false;"
}

trap release_lock EXIT
acquire_lock

# Get current version
CURRENT=$(psql "$DB_URL" -t -c "SELECT MAX(version) FROM schema_migrations;")

# Apply pending migrations
for file in $(ls "$MIGRATIONS_DIR"/*.up.sql | sort); do
    version=$(basename "$file" | cut -d_ -f1)
    if [[ "$version" -gt "${CURRENT:-0}" ]]; then
        echo "Applying: $file"
        psql "$DB_URL" -f "$file"
        psql "$DB_URL" -c "INSERT INTO schema_migrations (version) VALUES ($version);"
    fi
done
```

---

### Q13: Disaster Recovery Design (RTO/RPO) Level: SDE 3

**Question:** Design DR strategy for a payment system.

| Tier | RPO | RTO | Strategy |
|------|-----|-----|----------|
| Critical (payments DB) | 0 | <5 min | Synchronous replication + auto-failover |
| Important (user data) | <1 min | <15 min | Async replication + manual promotion |
| Standard (logs) | <1 hr | <4 hr | Cross-region backup + restore |

**Key components:**
- Active-standby in different AZ (same region): sync replication, <5min RTO.
- Cross-region DR: async replication, pilot-light (minimal infra running).
- Regular DR drills: Quarterly failover tests.
- Runbook: Automated decision tree for failure scenarios.

---

### Q14: Prometheus Alerting Rule Level: SDE 2

**Question:** Write alerting rules for a production service.

```yaml
groups:
  - name: api-server-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Error rate >1% for 5 minutes"
          dashboard: "https://grafana.internal/d/api-overview"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency >2s for 10 minutes"

      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} restarting frequently"
```

---

### Q15: Rollback Strategy for Failed Deploys Level: SDE 2

**Question:** Design automated rollback on deploy failure.

```bash
#!/bin/bash
set -euo pipefail

DEPLOY_TAG="$1"
NAMESPACE="production"
DEPLOYMENT="api-server"
TIMEOUT=300  # 5 minutes

echo "Deploying $DEPLOY_TAG..."
kubectl set image deployment/$DEPLOYMENT \
  app=registry.io/app:$DEPLOY_TAG -n $NAMESPACE

echo "Waiting for rollout..."
if ! kubectl rollout status deployment/$DEPLOYMENT \
  -n $NAMESPACE --timeout=${TIMEOUT}s; then
    echo "❌ Deploy failed. Rolling back..."
    kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE
    kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE --timeout=120s
    echo "✓ Rollback complete"
    exit 1
fi

# Post-deploy health check
sleep 30
ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=..." | jq '.data.result[0].value[1]')
if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
    echo "❌ Error rate too high ($ERROR_RATE). Rolling back..."
    kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE
    exit 1
fi

echo "✓ Deploy successful"
```

---

### Q16: Container Security Scanning Level: SDE 2

**Question:** Add security scanning to CI pipeline.

```yaml
# GitHub Actions with Trivy
security-scan:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Build image
      run: docker build -t app:scan .

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: app:scan
        format: table
        exit-code: 1
        severity: CRITICAL,HIGH
        ignore-unfixed: true

    - name: Run Trivy config scanner (Dockerfile)
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: config
        scan-ref: .
        exit-code: 1
```

---

### Q17: Observability Stack Design Level: SDE 3

**Question:** Design metrics + logs + traces for microservices.

**Architecture:**
```
Three Pillars:
1. Metrics: App → Prometheus → Grafana (dashboards + alerts)
2. Logs:    App → Fluentd → Loki (or ELK)
3. Traces:  App → OpenTelemetry SDK → Tempo/Jaeger

Correlation: Trace ID in logs + metrics labels → drill from alert to trace to log.
```

**Key decisions:** OpenTelemetry as single instrumentation layer. Trace-to-log correlation via trace_id. Service mesh (Istio) for network-level observability. Cost: sampling (1-10% traces), log retention tiers.

---

### Q18: Chaos Engineering Script Level: SDE 3

**Question:** Write a script to inject failure for resilience testing.

```bash
#!/bin/bash
# Kill random pod in a namespace
NAMESPACE="${1:-production}"
DURATION="${2:-60}"

echo "🔥 Starting chaos experiment in $NAMESPACE for ${DURATION}s"

# Select random pod
POD=$(kubectl get pods -n $NAMESPACE -o name | shuf -n 1)
echo "Target: $POD"

# Kill it
kubectl delete $POD -n $NAMESPACE --grace-period=0 --force

echo "Pod terminated. Monitoring recovery..."
sleep 10

# Verify recovery
READY=$(kubectl get pods -n $NAMESPACE --field-selector=status.phase=Running | wc -l)
DESIRED=$(kubectl get deployment -n $NAMESPACE -o jsonpath='{.items[0].spec.replicas}')

if [[ $READY -ge $DESIRED ]]; then
    echo "✓ Service recovered. Pods: $READY/$DESIRED"
else
    echo "⚠ Recovery incomplete. Pods: $READY/$DESIRED"
fi
```

**Follow-up:** Use LitmusChaos or Chaos Mesh for more sophisticated experiments (network partition, CPU stress, disk fill).

---

### Q19: Feature Flag Deployment Pipeline Level: SDE 3

**Question:** Design deployment flow where features are decoupled from releases.

```
Code Deploy (dark launch) → Feature Flag OFF → Integration Test
     → Gradual rollout: 1% → 5% → 25% → 100%
     → Monitor per-flag metrics
     → Stale flag cleanup (automated after 30 days at 100%)
```

**Key decisions:** Deploy != release. Ship code behind flags. Measure per-flag impact. Remove flags within sprint of 100% rollout (tech debt).

---

### Q20: Load Test Script (k6) Level: SDE 2

**Question:** Write a load test for an API.

```javascript
// k6 load test
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '1m', target: 50 },   // ramp up
    { duration: '5m', target: 200 },  // sustained load
    { duration: '2m', target: 500 },  // spike
    { duration: '1m', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    errors: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://api.example.com/users', {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  }) || errorRate.add(1);

  sleep(Math.random() * 2);
}
```

---

### Q21: Dependency Update Automation Level: SDE 2

**Question:** Design Dependabot/Renovate config for safe auto-updates.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: gomod
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    groups:
      minor-and-patch:
        update-types: ["minor", "patch"]
    labels: ["dependencies"]
    reviewers: ["platform-team"]
    commit-message:
      prefix: "deps:"

  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly
    labels: ["docker"]
```

**Key decisions:** Group minor+patch together. Pin major versions (breaking changes need human review). Auto-merge patch if CI passes. Security updates: immediate PR.

---

### Q22: Infrastructure Drift Detection Level: SDE 3

**Question:** Write a drift detection script for Terraform.

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${1:-production}"
SLACK_WEBHOOK="${SLACK_DRIFT_WEBHOOK}"

cd "terraform/environments/$WORKSPACE"
terraform init -backend=true -input=false > /dev/null 2>&1

echo "Running plan to detect drift..."
PLAN_OUTPUT=$(terraform plan -detailed-exitcode -input=false 2>&1) || EXIT_CODE=$?

case ${EXIT_CODE:-0} in
  0)
    echo "✓ No drift detected"
    ;;
  2)
    echo "⚠ Drift detected!"
    DRIFT_SUMMARY=$(echo "$PLAN_OUTPUT" | grep -E "^  [~+-]" | head -20)
    curl -s -X POST "$SLACK_WEBHOOK" \
      -H 'Content-type: application/json' \
      -d "{\"text\": \"🚨 Infrastructure drift in $WORKSPACE:\\n\`\`\`$DRIFT_SUMMARY\`\`\`\"}"
    ;;
  1)
    echo "❌ Terraform plan failed"
    exit 1
    ;;
esac
```

---

### Q23: Artifact Management Strategy Level: SDE 2

**Question:** Design artifact versioning and retention.

**Strategy:**
```
Tagging scheme: <service>:<semver>-<git-sha-short>
  e.g., api:1.2.3-abc1234

Retention policy:
  - Production tags: Keep forever
  - Release candidates: 30 days
  - Feature branch builds: 7 days
  - PR builds: 3 days

Storage: ECR/GCR with lifecycle policies
Promotion: dev → staging → production (same image, different tag alias)
SBOM: Generate and attach software bill of materials per artifact
```

---

### Q24: Git Hook Prevent Large File Commits Level: SDE 1

**Question:** Write a pre-commit hook that blocks large files.

```bash
#!/bin/bash
# .git/hooks/pre-commit
MAX_SIZE_KB=500
BLOCKED=0

while IFS= read -r file; do
    size_kb=$(du -k "$file" 2>/dev/null | awk '{print $1}')
    if [[ ${size_kb:-0} -gt $MAX_SIZE_KB ]]; then
        echo "❌ BLOCKED: $file (${size_kb}KB > ${MAX_SIZE_KB}KB limit)"
        BLOCKED=1
    fi
done < <(git diff --cached --name-only --diff-filter=ACM)

if [[ $BLOCKED -eq 1 ]]; then
    echo ""
    echo "Use 'git lfs track' for large files, or reduce file size."
    echo "To bypass: git commit --no-verify"
    exit 1
fi
```

---

### Q25: Self-Service Developer Platform Design Level: SDE 4

**Question:** Design an internal developer platform (IDP).

**Architecture:**
```
Developer Portal (Backstage/custom)
  ├── Service Catalog (ownership, docs, runbooks)
  ├── Template Engine (scaffold new services in <5 min)
  ├── CI/CD Dashboard (build status, deploy history)
  ├── Infrastructure Self-Service
  │   ├── Request database (form → Terraform apply)
  │   ├── Request cache cluster
  │   └── Request domain/cert
  ├── Observability Hub (unified metrics/logs/traces)
  └── Compliance Dashboard (security scan results, SLA tracking)

Principles:
  - Golden paths (opinionated defaults, escape hatches)
  - Everything as code (GitOps)
  - Guardrails, not gates (automated policy checks, not ticket queues)
  - Self-service with audit trail
```

**Key decisions:** Backstage for portal. Crossplane/Terraform for infrastructure. ArgoCD for GitOps. OPA/Kyverno for policy. Goal: <15 min from "I need a new service" to "it's deployed with observability."
