# Infrastructure & CI/CD Rubric

## 4-Tier Scorecard

| Dimension | 1 Strong No-Hire | 2 No-Hire | 3 Lean Hire | 4 Strong Hire |
|-----------|---------------------|-------------|---------------|-----------------|
| **Observability** | No monitoring awareness | Knows logs exist; no structured approach | Metrics + logs + basic alerting; Grafana/Prometheus | Distributed tracing; SLO-based alerts; correlation IDs; runbooks |
| **IaC** | Manual infrastructure | Scripts but not idempotent | Terraform/Pulumi with state management; modules | Multi-env promotion; drift detection; policy-as-code (OPA) |
| **Containerization** | Cannot explain Docker basics | Writes Dockerfiles but no optimization | Multi-stage builds; layer caching; security scanning | Distroless images; resource limits; runtime security (seccomp) |
| **Orchestration** | No awareness | Knows K8s exists | Deployments, services, HPA, configmaps | Operators; service mesh; multi-cluster; GitOps (ArgoCD/Flux) |
| **Deployment Strategy** | "Copy files to server" | Blue-green concept but no implementation | Canary with metrics gates; rollback automation | Progressive delivery; feature flags; traffic splitting by header |
| **Pipeline Design** | No CI/CD experience | Linear pipeline; no parallelism | Parallel stages; caching; quality gates | DAG-based; monorepo-aware; dynamic pipelines; self-healing |
| **Security** | Secrets in code | Vault/secrets manager awareness | SAST/DAST in pipeline; dependency scanning | Supply chain security (SLSA); signed artifacts; SBOM generation |

## Observability Stack Expectations

| Component | SDE 2 Bar | SDE 3 Bar | SDE 4 Bar |
|-----------|-----------|-----------|-----------|
| Metrics | Knows Prometheus/CloudWatch; basic dashboards | Custom metrics; histogram vs gauge; cardinality control | SLO/SLI framework; error budgets; capacity forecasting |
| Logs | Structured JSON logging; log levels | Correlation IDs; log aggregation (ELK/Loki) | Log-based alerting; sampling strategies; cost control |
| Traces | Aware of concept | OpenTelemetry instrumentation; span context | Cross-service trace analysis; tail-based sampling |
| Alerting | Static thresholds | Multi-window burn rate | Alert routing; escalation policies; noise reduction |

---

## Problem: Design a CI/CD Pipeline for a Monorepo (5 Services)

**Level:** SDE 3 | **Time:** 25 min

### Context
- Monorepo with 5 microservices: `auth`, `users`, `orders`, `payments`, `notifications`
- Shared libraries: `common-lib`, `proto-definitions`
- Stack: Go services, PostgreSQL, Redis, Kafka
- Target: Deploy to Kubernetes (EKS); 3 environments (dev/staging/prod)

### Expected Pipeline Architecture

```
┌─────────────────── Trigger (PR or merge to main) ──────────────────┐
│                                                                      │
▼                                                                      │
┌──────────────────┐                                                   │
│  Change Detection │ ← affected_services = diff-based (Nx/Turborepo) │
│  (What changed?)  │                                                   │
└────────┬─────────┘                                                   │
         │                                                             │
         ▼ (parallel per affected service)                             │
┌────────────────────────────────────────────────────┐                │
│  ┌─────────┐  ┌──────────┐  ┌────────┐  ┌──────┐ │                │
│  │  Lint   │  │  Build   │  │  Test  │  │ SAST │ │  ← Quality Gate│
│  │(golangci)│  │(Go build)│  │(unit+  │  │(Snyk)│ │                │
│  └─────────┘  └──────────┘  │ integ) │  └──────┘ │                │
│                              └────────┘            │                │
└────────────────────────┬───────────────────────────┘                │
                         │ All pass                                    │
                         ▼                                             │
┌────────────────────────────────────────────────────┐                │
│  Container Build & Push                             │                │
│  - Multi-stage Dockerfile (builder → distroless)    │                │
│  - Tag: git-sha + semver                           │                │
│  - Push to ECR; sign with Cosign                   │                │
└────────────────────────┬───────────────────────────┘                │
                         │                                             │
         ┌───────────────┼───────────────┐                            │
         ▼               ▼               ▼                            │
    ┌─────────┐    ┌──────────┐    ┌──────────┐                      │
    │   Dev   │───▶│ Staging  │───▶│   Prod   │                      │
    │(auto)   │    │(auto+E2E)│    │(manual+  │                      │
    │         │    │          │    │ canary)  │                      │
    └─────────┘    └──────────┘    └──────────┘                      │
                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

| Decision | Options | Recommended | Reasoning |
|----------|---------|-------------|-----------|
| Change detection | Full rebuild vs diff-based | Diff-based (git diff + dep graph) | 5x faster; only build what changed |
| Shared lib change | Rebuild dependents vs skip | Rebuild all consumers | Shared lib = high blast radius |
| Test strategy | Unit only vs full E2E | Unit in PR; E2E in staging | Fast feedback + confidence |
| Image tagging | Latest vs SHA vs semver | SHA + semver on release | Immutable; traceable; rollback-friendly |
| Prod deployment | Big-bang vs canary | Canary (10% → 50% → 100%) | Risk mitigation with metrics gates |
| Rollback | Redeploy previous vs revert | Redeploy previous SHA | Faster than git revert + rebuild |

### Quality Gates

| Gate | Stage | Criteria | Block on Fail? |
|------|-------|----------|----------------|
| Lint | PR | Zero warnings | Yes |
| Unit tests | PR | 100% pass; >80% coverage | Yes |
| SAST | PR | No critical/high CVEs | Yes |
| Integration tests | Staging | All contract tests pass | Yes |
| E2E smoke | Staging | Critical paths pass | Yes |
| Canary health | Prod | Error rate <0.1%; p99 <500ms | Yes (auto-rollback) |

### Follow-up Questions
1. How do you handle database migrations in this pipeline?
2. What's your strategy for secrets rotation without downtime?
3. How do you test Kafka consumer/producer contracts across services?
4. How would you add preview environments for each PR?
5. What metrics would you use to measure pipeline health itself?

### Scoring Guide

| Score | Signal |
|-------|--------|
| 1 | Manual builds; no pipeline concept |
| 2 | Linear pipeline; full rebuild; no environments |
| 3 | Diff-based builds; multi-env; quality gates; canary concept |
| 4 | Full architecture with monorepo awareness; progressive delivery; observability of pipeline itself |
