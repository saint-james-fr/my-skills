# Microservices & Infrastructure — Detailed Reference

Companion to `SKILL.md`. Deep dives on Netflix architecture, Kubernetes tooling, CI/CD pipelines, service mesh internals, container orchestration, deployment strategies, and Reddit's architecture.

---

## 1. Netflix Architecture Case Study

### Tech Stack Overview

| Layer | Technology |
|-------|-----------|
| Mobile | Swift (iOS), Kotlin (Android) |
| Web | React |
| Frontend ↔ Server | GraphQL (formerly REST via Zuul) |
| Backend | Java (Spring Boot), Netflix OSS |
| API Gateway | Zuul (now proxy role only) |
| Service Discovery | Eureka |
| Databases | EV Cache, Cassandra, CockroachDB |
| Messaging | Apache Kafka, Flink |
| Video Storage | S3, Open Connect (custom CDN) |
| Data Processing | Flink, Spark → Tableau; Redshift for structured warehouse |
| CI/CD | JIRA, Confluence, Jenkins, Gradle, Spinnaker, Atlas |
| Chaos Engineering | Chaos Monkey (part of Simian Army) |
| Monitoring | PagerDuty, Atlas (custom metrics platform) |

### API Architecture Evolution

Netflix's API layer evolved through three major iterations, each solving problems introduced by the previous approach.

#### Iteration 1: API Gateway (Zuul)

```
┌──────────┐     ┌──────┐     ┌────────────────────────┐
│ Client   │────▶│ Zuul │────▶│ Microservice A, B, C…  │
│ (TV/Web/ │     │ (GW) │     │ (Java 8)               │
│  Mobile) │     └──────┘     └────────────────────────┘
└──────────┘
```

- Every piece of functionality owned by a Java microservice
- Rendering one screen (e.g., List of Lists of Movies — LOLOMO) required fetching data from 10s of microservices
- Single gateway orchestrated fanout to backend services
- **Problem**: One gateway for all client types created a bottleneck; each client (TV, mobile, web) had subtly different requirements

#### Iteration 2: Backend for Frontend (BFF) with Groovy + RxJava

```
┌─────┐   ┌──────────┐   ┌───────────┐
│ TV  │──▶│ BFF (TV) │──▶│           │
├─────┤   ├──────────┤   │ Services  │
│ Web │──▶│ BFF (Web)│──▶│ (Java 8)  │
├─────┤   ├──────────┤   │           │
│ iOS │──▶│ BFF (iOS)│──▶│           │
└─────┘   └──────────┘   └───────────┘
              │
         Groovy scripts
         RxJava for fanout
```

- Zuul moved to a pure proxy role
- Each frontend got its own mini backend (BFF pattern)
- BFFs built with Groovy scripts; service fanout via RxJava for thread management
- **Problem**: Required UI developers to write Groovy scripts; reactive programming is inherently complex

#### Iteration 3: GraphQL Federation (Current)

```
┌─────────┐    ┌──────────────────┐    ┌───────────────────────┐
│ Client  │───▶│ GraphQL Gateway  │───▶│ Domain Graph Services │
│         │    │ (Federation)     │    │ (DGS)                 │
└─────────┘    └──────────────────┘    │ Java 17, Spring Boot 3│
                                       │ Netflix OSS packages  │
                                       └───────────────────────┘
```

- Client specifies exactly which fields it needs → solves over/underfetching
- GraphQL Federation calls the necessary microservices (Domain Graph Services — DGS)
- DGS built with Java 17, Spring Boot 3, and Netflix OSS packages
- Java 8 → Java 17 migration yielded **20% CPU gains**
- Now migrating to Java 21 to leverage virtual threads

### Key Architectural Lessons from Netflix

| Lesson | Detail |
|--------|--------|
| API Gateway is step 1, not the end | Start with Zuul-style gateway, evolve to BFF or GraphQL Federation |
| BFF pattern solves multi-client divergence | But shifts complexity to the BFF layer |
| GraphQL Federation is the current best practice | Declarative data fetching, strong typing, schema stitching |
| Chaos Engineering is essential | Chaos Monkey randomly kills services in production to test resilience |
| Observability is non-negotiable | Atlas for metrics, extensive logging, distributed tracing |
| Java remains dominant at scale | Netflix proves JVM performance at massive scale; virtual threads are the future |

---

## 2. Kubernetes Tools Stack Wheel

The K8s ecosystem is vast. This categorizes the major tool families.

| Category | Tools | Purpose |
|----------|-------|---------|
| **Cluster Management** | kOps, Rancher, Kubespray | Bootstrap and manage clusters |
| **Package Management** | Helm, Kustomize | Template and deploy K8s manifests |
| **CI/CD** | ArgoCD, Flux, Jenkins X, Tekton | GitOps-based deployment |
| **Service Mesh** | Istio, Linkerd, Consul Connect | mTLS, traffic management, observability |
| **Monitoring** | Prometheus, Grafana, Thanos | Metrics collection and visualization |
| **Logging** | EFK (Elasticsearch+Fluentd+Kibana), Loki | Log aggregation |
| **Tracing** | Jaeger, Zipkin, Tempo | Distributed tracing |
| **Security** | Falco, OPA/Gatekeeper, Trivy | Runtime security, policy enforcement, image scanning |
| **Networking** | Calico, Cilium, Flannel | CNI plugins for pod networking |
| **Storage** | Rook-Ceph, Longhorn, CSI drivers | Persistent storage orchestration |
| **Autoscaling** | KEDA, Cluster Autoscaler, VPA | Scale pods and nodes based on demand |
| **Ingress** | Nginx Ingress, Traefik, Ambassador | HTTP/HTTPS traffic routing |
| **Secrets** | Sealed Secrets, External Secrets, Vault | Secure secret management |
| **Cost** | Kubecost, OpenCost | Cluster cost analysis and optimization |

### Choosing Your Stack (Pragmatic Defaults)

For a team starting with K8s, a solid default stack:

```
Ingress:       Nginx Ingress Controller
Package Mgmt:  Helm
CI/CD:         ArgoCD (GitOps)
Monitoring:    Prometheus + Grafana
Logging:       Loki + Grafana
Tracing:       Jaeger (or Tempo)
Service Mesh:  Start without one; add Linkerd when needed
Security:      Trivy for image scanning, OPA for policies
Secrets:       External Secrets Operator + Vault
```

---

## 3. CI/CD Pipeline — Detailed Stages

### Full Pipeline Flow

```
┌────────┐   ┌───────┐   ┌──────────────┐   ┌─────────┐   ┌──────────┐
│ Source │──▶│ Build │──▶│ Test         │──▶│ Package │──▶│ Artifact │
│ (Git)  │   │       │   │              │   │ (Docker)│   │ Registry │
└────────┘   └───────┘   └──────────────┘   └─────────┘   └──────────┘
                              │                                  │
                    ┌─────────┴──────────┐                       │
                    │ Unit  Integration  │                       │
                    │ E2E   Security     │                       │
                    │ Coverage gates     │                       ▼
                                                          ┌──────────┐
                    ┌──────────────────────────────────────│  Deploy  │
                    │                                     │  (Dev)   │
                    │                                     └────┬─────┘
                    │                                          │
               ┌────▼─────┐  ┌──────────┐  ┌──────────┐  ┌───▼──────┐
               │ QA1/QA2  │─▶│  UAT     │─▶│ Release  │─▶│Production│
               │ (Feature │  │ (Staging)│  │ Candidate│  │          │
               │  testing)│  └──────────┘  └──────────┘  └──────────┘
               └──────────┘
```

### Stage Details

| Stage | Activities | Tools | Quality Gates |
|-------|-----------|-------|---------------|
| **Source** | PR created, code review, linting | Git, GitHub/GitLab | Branch protection, ≥1 approval |
| **Build** | Compile, resolve dependencies, transpile | Gradle, Maven, npm, esbuild | Build must succeed |
| **Static Analysis** | Lint, type-check, code quality | SonarQube, ESLint, Semgrep | Quality gate thresholds |
| **Unit Tests** | Isolated function/class tests | Jest, JUnit, pytest | Coverage ≥ 80% |
| **Integration Tests** | Service-to-service, DB interactions | Testcontainers, Supertest | All pass |
| **Security Scan** | Dependency vulnerabilities, SAST | Snyk, Trivy, Dependabot | No critical/high CVEs |
| **Package** | Build Docker image, tag with Git SHA | Docker, Buildpacks | Image scan passes |
| **Deploy to Dev** | Auto-deploy to development environment | Helm, ArgoCD | Smoke tests pass |
| **QA Testing** | Manual + automated QA, regression | Cypress, Playwright, QA team | Test suite passes |
| **UAT** | Business stakeholder validation | UAT environment | Sign-off |
| **Production Deploy** | Canary/Blue-Green/Rolling deployment | Spinnaker, ArgoCD, Flux | Health checks pass |
| **Post-Deploy** | Monitor, alert, rollback if needed | Prometheus, PagerDuty, Sentry | Error rate < threshold |

---

## 4. Service Mesh Internals

### What Is a Service Mesh?

A dedicated infrastructure layer that handles service-to-service communication. It extracts networking concerns (retries, mTLS, observability, traffic shaping) from application code into sidecar proxies.

### Architecture: Data Plane + Control Plane

```
┌─────────────────────────────────────────────────┐
│                 Control Plane                    │
│  ┌───────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Pilot │  │ Citadel  │  │ Policy/Telemetry │  │
│  │(config│  │(cert mgmt│  │ (Mixer/WASM)     │  │
│  │ + SD) │  │ + mTLS)  │  │                  │  │
│  └───┬───┘  └────┬─────┘  └────────┬─────────┘  │
│      │           │                 │             │
└──────┼───────────┼─────────────────┼─────────────┘
       │           │                 │
┌──────┼───────────┼─────────────────┼─────────────┐
│      ▼           ▼                 ▼  Data Plane │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │ Pod A     │  │ Pod B     │  │ Pod C     │    │
│  │ ┌──────┐  │  │ ┌──────┐  │  │ ┌──────┐  │    │
│  │ │ App  │  │  │ │ App  │  │  │ │ App  │  │    │
│  │ └──┬───┘  │  │ └──┬───┘  │  │ └──┬───┘  │    │
│  │ ┌──▼───┐  │  │ ┌──▼───┐  │  │ ┌──▼───┐  │    │
│  │ │Envoy │  │  │ │Envoy │  │  │ │Envoy │  │    │
│  │ │proxy │◄─┼──┼─│proxy │◄─┼──┼─│proxy │  │    │
│  │ └──────┘  │  │ └──────┘  │  │ └──────┘  │    │
│  └───────────┘  └───────────┘  └───────────┘    │
└──────────────────────────────────────────────────┘
```

### Istio vs Linkerd

| Feature | Istio | Linkerd |
|---------|-------|---------|
| Proxy | Envoy (C++) | linkerd2-proxy (Rust) |
| Complexity | High — many features, many CRDs | Low — lightweight, focused |
| Performance overhead | ~2-5ms per hop | ~1ms per hop |
| mTLS | Yes, automatic | Yes, automatic |
| Traffic management | Advanced (fault injection, mirroring, A/B) | Basic (retries, timeouts, splits) |
| Observability | Full (metrics, traces, logs) | Metrics + golden signals |
| Multi-cluster | Yes | Yes |
| WASM extensibility | Yes | No |
| Best for | Complex multi-team, advanced traffic control | Simplicity, performance-sensitive |

### When to Introduce a Service Mesh

| Signal | Action |
|--------|--------|
| < 5 services | Probably don't need one; use simple HTTP + retry libraries |
| 5-20 services | Consider Linkerd for mTLS and observability with minimal overhead |
| > 20 services, multiple teams | Istio for full traffic management, security policies, canary releases |
| Already have strong observability | May only need mTLS → Linkerd |

---

## 5. Container Orchestration Comparison

| Feature | Docker Compose | Docker Swarm | Kubernetes | AWS ECS | Nomad |
|---------|---------------|-------------|------------|---------|-------|
| Complexity | Very low | Low | High | Medium | Medium |
| Scaling | Manual | Auto (basic) | Auto (advanced) | Auto | Auto |
| Service discovery | DNS | Built-in | CoreDNS + Services | Built-in | Consul |
| Networking | Bridge, overlay | Overlay | CNI plugins | VPC | CNI/bridge |
| Storage | Volumes | Volumes | PV/PVC, CSI | EBS/EFS | CSI |
| Health checks | Basic | Basic | Liveness/Readiness/Startup | ELB health | Task checks |
| Rolling updates | No | Yes | Yes (configurable) | Yes | Yes |
| Secrets mgmt | Docker secrets | Docker secrets | K8s Secrets + external | SSM/Secrets Manager | Vault |
| Best for | Local dev, small deployments | Small clusters | Production at scale | AWS-native workloads | Multi-runtime (containers + VMs) |

### Decision Matrix

```
Local dev / PoC             → Docker Compose
Small team, < 10 services   → ECS Fargate or Railway/Fly.io
AWS-native, managed          → ECS or EKS
Multi-cloud / large scale   → Kubernetes (self-managed or managed)
Mixed workloads (VM + container) → Nomad
```

---

## 6. Deployment Strategies — Detailed Comparison

### Rolling Deployment

```
Time 0:  [v1] [v1] [v1] [v1]
Time 1:  [v2] [v1] [v1] [v1]   ← replace one at a time
Time 2:  [v2] [v2] [v1] [v1]
Time 3:  [v2] [v2] [v2] [v1]
Time 4:  [v2] [v2] [v2] [v2]   ← complete
```

- **Pros**: Zero downtime, resource-efficient (no extra infra)
- **Cons**: Both versions run simultaneously (compatibility required), slow rollback
- K8s config: `maxUnavailable: 1, maxSurge: 1` in Deployment spec

### Blue-Green Deployment

```
              ┌─────────────┐
   Traffic ──▶│ Load        │
              │ Balancer    │
              └──────┬──────┘
                     │
            ┌────────┴────────┐
            │                 │
     ┌──────▼──────┐   ┌─────▼───────┐
     │  Blue (v1)  │   │ Green (v2)  │
     │  [active]   │   │  [standby]  │
     └─────────────┘   └─────────────┘
            │                 │
            └── switch ───────┘
```

- **Pros**: Instant rollback (switch back), full testing of new version before traffic
- **Cons**: 2x infrastructure cost, database schema must be backward-compatible
- Tools: Spinnaker, AWS CodeDeploy, manual DNS/LB switch

### Canary Deployment

```
                    ┌─────────┐
     Traffic ──────▶│  LB     │
                    └────┬────┘
                         │
              ┌──────────┴──────────┐
              │ 95%                 │ 5%
       ┌──────▼──────┐      ┌──────▼──────┐
       │  Stable v1  │      │ Canary v2   │
       │  (most pods)│      │ (few pods)  │
       └─────────────┘      └─────────────┘
```

- **Pros**: Minimal blast radius, real production traffic validation
- **Cons**: Requires traffic splitting (Istio, Nginx, Envoy), monitoring to detect issues
- Progression: 5% → 25% → 50% → 100% with automated rollback on error spike

### Feature Flags (Complementary)

Not a deployment strategy per se, but complementary: deploy code dark (disabled), then enable per user/segment via feature flag service (LaunchDarkly, Unleash, Flipt). Decouples deployment from release.

---

## 7. Reddit Core Architecture Analysis

Based on Reddit engineering blog research. Architecture evolves continuously.

### Architecture Overview

```
┌──────────┐     ┌───────┐     ┌─────────┐     ┌──────────────────┐
│  Users   │────▶│ CDN   │────▶│  Load   │────▶│  Application     │
│          │     │(Fastly)│     │Balancer │     │  Services        │
└──────────┘     └───────┘     └─────────┘     └────────┬─────────┘
                                                        │
                                    ┌───────────────────┼───────────────┐
                                    │                   │               │
                              ┌─────▼─────┐     ┌──────▼──────┐  ┌────▼─────┐
                              │ GraphQL   │     │  Go         │  │ Python   │
                              │ Federation│     │ Microservices│  │ Monolith │
                              │ (Gateway) │     │             │  │ (legacy) │
                              └─────┬─────┘     └──────┬──────┘  └────┬─────┘
                                    │                   │              │
                         ┌──────────┼──────────────────┼──────────────┼────┐
                         │          │                   │              │    │
                   ┌─────▼──┐  ┌───▼────┐  ┌──────────▼┐  ┌─────────▼┐   │
                   │Postgres│  │memcached│  │ Cassandra │  │RabbitMQ  │   │
                   │        │  │        │  │           │  │+ Kafka   │   │
                   └────────┘  └────────┘  └───────────┘  └──────────┘   │
                                                                         │
                                                              ┌──────────▼──┐
                                                              │ AWS + K8s   │
                                                              │ Spinnaker   │
                                                              │ Drone CI    │
                                                              │ Terraform   │
                                                              └─────────────┘
```

### Key Architecture Decisions

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| CDN | Fastly | Front for the entire application; edge caching |
| Frontend | TypeScript, Node.js frameworks | Started with jQuery (2009), modernized over time |
| Mobile | Native Android + iOS apps | Platform-native experience |
| Backend (new) | Go microservices | Performance, concurrency model |
| Backend (legacy) | Python monolith | Original application; gradually migrating |
| API Layer | GraphQL Federation | Combined smaller GraphQL APIs (Domain Graph Services) |
| Core DB | PostgreSQL | Core data model |
| Cache | Memcached | Sits in front of Postgres to reduce load |
| NoSQL | Cassandra | New features; chosen for resiliency and availability |
| CDC | Debezium | Change Data Capture for replication and cache consistency |
| Async Jobs | RabbitMQ | Expensive operations (voting, link submission) deferred |
| Streaming | Kafka | Real-time content safety checks and moderation |
| Hosting | AWS + Kubernetes | All apps and internal services |
| Deployment | Spinnaker, Drone CI, Terraform | CI/CD and infrastructure as code |

### Evolution Timeline

| Year | Change |
|------|--------|
| 2005 | Launched as Python (Pylons) monolith |
| 2009 | Added jQuery for frontend interactivity |
| ~2017 | Started migrating to TypeScript frontend |
| ~2018 | Go microservices for new backend services |
| 2021 | Started GraphQL Federation |
| 2022 | Split GraphQL monolith into Go subgraphs for core entities |
| Present | Hybrid: Python monolith + Go microservices, GraphQL Federation |

### Lessons from Reddit's Architecture

| Lesson | Detail |
|--------|--------|
| Strangler Fig in action | Python monolith gradually replaced by Go microservices |
| GraphQL Federation scales API complexity | Domain Graph Services allow independent team ownership |
| Debezium for cache consistency | CDC solves the cache invalidation problem at scale |
| Separate sync vs async paths | User-facing actions use sync path; expensive ops deferred to RabbitMQ |
| Kafka for real-time safety | Content moderation runs as stream processing |
| K8s as the platform | Unifies hosting for all applications and internal services |

---

## 8. DevSecOps Integration

DevSecOps integrates security into every stage of the CI/CD pipeline rather than treating it as a gate at the end.

| Stage | Security Activity | Tools |
|-------|-------------------|-------|
| Code | SAST (static analysis), secret scanning | SonarQube, Semgrep, GitLeaks |
| Dependencies | SCA (software composition analysis) | Snyk, Dependabot, npm audit |
| Build | Image scanning, SBOM generation | Trivy, Grype, Syft |
| Deploy | Policy enforcement, admission control | OPA/Gatekeeper, Kyverno |
| Runtime | DAST, runtime protection, anomaly detection | Falco, Aqua Security |
| Infrastructure | IaC scanning, compliance checks | Checkov, tfsec, Bridgecrew |

### Infrastructure as Code (IaC) vs Configuration Management

| Aspect | IaC | Configuration Management |
|--------|-----|-------------------------|
| Focus | Provisioning infrastructure | Maintaining desired state of existing systems |
| Approach | Declarative (what, not how) | Can be declarative or imperative |
| Scope | Servers, networks, storage, services | Software packages, files, services on existing servers |
| Tools | Terraform, Pulumi, CloudFormation | Ansible, Chef, Puppet, Salt |
| Idempotency | Built-in (desired state) | Depends on tool and implementation |
| Best for | Creating/destroying entire environments | Configuring software on existing infrastructure |
