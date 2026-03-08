---
name: system-design-microservices-infra
description: Microservices architecture and infrastructure — microservice design patterns, Docker and Kubernetes, DevOps practices, CI/CD, architectural styles, observability, and production-readiness. Use when designing microservice architectures, containerizing applications, setting up CI/CD pipelines, choosing architectural patterns, or implementing observability.
---

# Microservices & Infrastructure

## 1. Microservice Architecture

### What Makes a Good Microservice

| Principle | Description |
|-----------|-------------|
| Single responsibility | Does one thing well, aligned to a bounded context |
| Owned data | Each service owns its database/schema — no shared DB |
| Independent deployment | Deploy without coordinating with other services |
| Lightweight communication | REST, gRPC, or async messaging — no tight coupling |
| Stateless | No local session state; externalize to Redis/DB |
| Autonomous team | One team owns the service end-to-end |

### 9 Best Practices for Building Microservices

| # | Practice | Key Point |
|---|----------|-----------|
| 1 | Design for failure | Circuit breakers, bulkheads, graceful degradation |
| 2 | Build small services | One service = one business capability |
| 3 | Lightweight protocols | REST, gRPC, or message brokers for communication |
| 4 | Service discovery | Consul, Eureka, or Kubernetes Services |
| 5 | Data ownership | Each service owns its data; reduce coupling |
| 6 | Resiliency patterns | Retry policies, caching, rate limiting |
| 7 | Security at all levels | Secure every communication path; mTLS between services |
| 8 | Centralized logging | Aggregate logs across all services (ELK, Loki) |
| 9 | Containerization | Docker + K8s for isolation, scaling, and deployment |

### 9 Essential Components of a Production Microservice

| Component | Purpose | Example Tools |
|-----------|---------|---------------|
| API Gateway | Unified entry point, routing, filtering, rate limiting | Kong, AWS API Gateway, Envoy |
| Service Registry | Service discovery and health tracking | Consul, Eureka, Zookeeper |
| Service Layer | Business logic; each service runs on multiple instances | Spring Boot, NestJS, Express |
| Authorization Server | Identity and access control | Keycloak, Azure AD, Okta |
| Data Storage | Persistent application data | PostgreSQL, MySQL, MongoDB |
| Distributed Cache | Boost read performance | Redis, Memcached, Couchbase |
| Async Communication | Decouple services, event-driven flows | Kafka, RabbitMQ, NATS |
| Metrics & Visualization | Monitor service health and performance | Prometheus + Grafana |
| Log Aggregation | Centralized log collection and search | ELK Stack (Elasticsearch, Logstash, Kibana) |

## 2. Architectural Patterns

### Monolith vs SOA vs Microservices

| Dimension | Monolith | SOA | Microservices |
|-----------|----------|-----|---------------|
| Deployment | Single unit | Service-level | Per-service |
| Data | Shared DB | Shared or per-service | Per-service (strict) |
| Communication | In-process | ESB / SOAP | REST / gRPC / async |
| Team structure | One large team | Multiple teams, shared governance | Small autonomous teams |
| Scaling | Scale entire app | Scale service groups | Scale individual services |
| Complexity | Low initially, grows fast | Medium | High upfront, manageable long-term |
| Best for | MVPs, small apps | Enterprise integration | Cloud-native, high-scale systems |

### Top 5 Architectural Patterns

| Pattern | Structure | Best For | Trade-off |
|---------|-----------|----------|-----------|
| **Layered** | Presentation → Business → Persistence → DB | Traditional web apps, CRUD | Simple but tight vertical coupling |
| **Event-Driven** | Producers → Event Bus → Consumers | Real-time systems, decoupled flows | Eventual consistency, harder debugging |
| **Microkernel** | Core system + plug-in modules | IDEs, browsers, extensible products | Plug-in management complexity |
| **Space-Based** | Processing units + virtualized middleware | High-volume concurrent users | Complex infrastructure, data consistency |
| **Pipe-and-Filter** | Source → Filter₁ → Filter₂ → Sink | Data processing, ETL, compilers | Not suited for interactive applications |

## 3. Architectural Styles

| Style | Structure | Data Flow | Best For |
|-------|-----------|-----------|----------|
| **MVC** | Model ↔ Controller ↔ View | Controller mediates | Web apps (Rails, Django, Spring MVC) |
| **MVP** | Model ↔ Presenter ↔ View | Presenter owns logic; View is passive | Android, testable UI |
| **MVVM** | Model ↔ ViewModel ↔ View | Two-way data binding | WPF, SwiftUI, Vue.js |
| **Hexagonal** | Core ← Ports → Adapters | Ports define boundaries | Domain-heavy services, DDD |
| **Event-Driven** | Producers → Broker → Consumers | Async events | Decoupled microservices, real-time |
| **CQRS** | Command side ≠ Query side | Separate write/read models | High-read systems, event sourcing |

Key insight: MVC is ~50 years old. Every pattern has a "View" for display and user input, a "Model" for business data, and a translator layer (Controller/Presenter/ViewModel) that mediates between them.

## 4. Docker

### Architecture

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│ Docker CLI  │────▶│ Docker      │────▶│ Docker       │
│ (client)    │     │ Daemon      │     │ Registry     │
└─────────────┘     │ (host)      │     │ (Hub/ECR/GCR)│
                    │ ┌─────────┐ │     └──────────────┘
                    │ │Container│ │
                    │ │Container│ │
                    │ └─────────┘ │
                    └─────────────┘
```

- **Client**: CLI that talks to the Docker daemon
- **Host/Daemon**: Manages images, containers, networks, and volumes
- **Registry**: Stores Docker images (Docker Hub, ECR, GCR)

### `docker run` Flow

1. Pull the image from the registry
2. Create a new container
3. Allocate a read-write filesystem to the container
4. Create a network interface (connects to default network)
5. Start the container

### Top 8 Docker Concepts

| Concept | Description |
|---------|-------------|
| Image | Read-only template with app code + dependencies |
| Container | Running instance of an image (isolated process) |
| Dockerfile | Build instructions for creating images |
| Volume | Persistent storage that survives container restarts |
| Network | Isolated communication between containers |
| Compose | Multi-container app definition (docker-compose.yml) |
| Layer caching | Each Dockerfile instruction creates a cacheable layer |
| Multi-stage build | Separate build and runtime stages to minimize image size |

### Dockerfile Best Practices

| Practice | Why |
|----------|-----|
| Use specific base image tags | Avoid `latest`; pin versions for reproducibility |
| Multi-stage builds | Separate build deps from runtime; smaller images |
| Order layers by change frequency | Put rarely-changing layers first for cache hits |
| Use `.dockerignore` | Exclude node_modules, .git, tests from build context |
| Run as non-root user | `USER node` — security best practice |
| One process per container | Easier to scale and debug |
| Health checks | `HEALTHCHECK CMD curl -f http://localhost/ || exit 1` |
| Minimize layer count | Combine related `RUN` commands with `&&` |

## 5. Kubernetes (K8s)

### Core Concepts

| Resource | Purpose |
|----------|---------|
| **Pod** | Smallest deployable unit; one or more containers sharing network/storage |
| **Service** | Stable network endpoint to access a set of Pods (ClusterIP, NodePort, LoadBalancer) |
| **Deployment** | Declarative Pod management; rolling updates, rollbacks |
| **ReplicaSet** | Ensures desired number of Pod replicas are running |
| **Namespace** | Logical cluster partitioning (dev, staging, prod) |
| **ConfigMap** | Externalized configuration as key-value pairs |
| **Secret** | Sensitive data (passwords, tokens) stored encoded |
| **Ingress** | HTTP/HTTPS routing rules to Services |
| **DaemonSet** | Runs one Pod per node (logging agents, monitoring) |
| **StatefulSet** | For stateful apps with stable network identity and persistent storage |

### Top 10 K8s Design Patterns

| # | Pattern | Use Case |
|---|---------|----------|
| 1 | Sidecar | Add logging/proxy to a Pod without modifying the main container |
| 2 | Ambassador | Proxy outbound connections (e.g., connect to different DB per env) |
| 3 | Adapter | Standardize heterogeneous output (e.g., normalize metrics format) |
| 4 | Init Container | Run setup tasks before the main container starts |
| 5 | Job / CronJob | Run batch or scheduled tasks to completion |
| 6 | DaemonSet | One agent per node (log collector, monitoring agent) |
| 7 | StatefulSet | Databases, Kafka brokers — need stable identity and storage |
| 8 | Operator | Automate complex application lifecycle (e.g., database failover) |
| 9 | Horizontal Pod Autoscaler | Auto-scale Pods based on CPU/memory/custom metrics |
| 10 | Pod Disruption Budget | Guarantee minimum availability during voluntary disruptions |

### K8s vs Simpler Alternatives

| Factor | Use K8s | Use simpler (ECS, Railway, Fly.io) |
|--------|---------|-----------------------------------|
| Team size | > 5 engineers, dedicated platform team | Small team, no dedicated ops |
| Services | > 10 microservices | < 5 services |
| Scale | Need auto-scaling, multi-region | Moderate, predictable traffic |
| Compliance | Need fine-grained network policies | Standard compliance |
| Learning curve | Team has K8s experience | Prefer managed simplicity |

## 6. DevOps & CI/CD

### CI/CD Pipeline Stages

```
Code → Build → Test → Package → Deploy to Staging → Deploy to Prod → Monitor
 │       │       │        │            │                  │              │
 Git   Compile  Unit    Docker      Smoke/QA         Blue-Green     Prometheus
 push  + Lint   + Int   image       + UAT            or Canary      + Grafana
                + E2E   to registry                  + Rolling
```

| Stage | Key Activities | Tools |
|-------|---------------|-------|
| Source | Commit, PR review, branch strategy | Git, GitHub, GitLab |
| Build | Compile, lint, dependency install | Gradle, Maven, npm, esbuild |
| Test | Unit, integration, E2E, security scan | Jest, JUnit, Cypress, SonarQube |
| Package | Build Docker image, store artifact | Docker, Artifactory, ECR |
| Deploy (staging) | Deploy to QA/UAT, run smoke tests | Jenkins, GitHub Actions, ArgoCD |
| Deploy (prod) | Release with strategy (canary, blue-green) | Spinnaker, Flux, ArgoCD |
| Monitor | Logs, metrics, alerts, on-call | Prometheus, Grafana, PagerDuty, Sentry |

### How Companies Ship Code to Production

1. Product owner creates user stories from requirements
2. Dev team picks stories into a 2-week sprint
3. Developers commit to Git
4. Build triggered (Jenkins/GH Actions) — unit tests, coverage gates, SonarQube
5. Artifact stored, deployed to dev environment
6. Features tested independently in QA1/QA2 environments
7. QA team runs regression + performance testing
8. Successful builds deployed to UAT
9. Release candidates deployed to production on schedule
10. SRE team monitors production

### GitOps Workflow

| Principle | Description |
|-----------|-------------|
| Git as single source of truth | All config (app + infra) lives in Git |
| Declarative state | Describe desired state, not imperative steps |
| Automated delivery | Git push triggers CI/CD → deploy |
| Immutable infrastructure | Never modify live; redeploy from Git |
| Observability & feedback | Continuously reconcile actual state with Git state |
| Audit trail | Every change is a Git commit with author and timestamp |

Tools: ArgoCD, Flux, Jenkins X

### Deployment Strategies

| Strategy | How It Works | Downtime | Risk | Rollback Speed |
|----------|-------------|----------|------|----------------|
| **Rolling** | Replace instances one by one | Zero | Medium — mixed versions briefly | Moderate |
| **Blue-Green** | Two identical envs; switch traffic at once | Zero | Low — full env tested before switch | Instant (switch back) |
| **Canary** | Route small % of traffic to new version | Zero | Lowest — gradual exposure | Fast (route back) |
| **Recreate** | Stop all old, start all new | Yes | High — all-or-nothing | Slow (redeploy old) |
| **A/B Testing** | Route by user segment (not version strategy per se) | Zero | Low | Fast |

## 7. 12-Factor App

| # | Factor | Description | Modern Interpretation |
|---|--------|-------------|----------------------|
| 1 | **Codebase** | One repo per app, tracked in version control | Monorepo or single-service repo; Git |
| 2 | **Dependencies** | Explicitly declare and isolate dependencies | package.json, go.mod, requirements.txt |
| 3 | **Config** | Store config in environment variables | .env files, ConfigMaps, Vault, AWS SSM |
| 4 | **Backing Services** | Treat DBs, queues, caches as attached resources | Swap Postgres for Aurora without code changes |
| 5 | **Build, Release, Run** | Strict separation of build, release, and run stages | Docker build → tag → deploy |
| 6 | **Processes** | Execute app as stateless processes | No sticky sessions; store state in Redis/DB |
| 7 | **Port Binding** | Export services via port binding | App self-contains its web server (no external Tomcat) |
| 8 | **Concurrency** | Scale out via the process model | Horizontal pod autoscaling in K8s |
| 9 | **Disposability** | Fast startup, graceful shutdown | SIGTERM handling, readiness/liveness probes |
| 10 | **Dev/Prod Parity** | Keep dev, staging, and prod as similar as possible | Docker Compose locally, same images in prod |
| 11 | **Logs** | Treat logs as event streams | Write to stdout; collect with Fluentd/Logstash |
| 12 | **Admin Processes** | Run admin tasks as one-off processes | K8s Jobs, database migrations via CI |

## 8. Observability (Three Pillars)

| Pillar | What It Captures | Tools |
|--------|-----------------|-------|
| **Logs** | Discrete events with context (timestamp, level, message) | ELK Stack, Loki + Grafana, CloudWatch Logs |
| **Metrics** | Numeric measurements over time (counters, gauges, histograms) | Prometheus, Datadog, CloudWatch Metrics |
| **Traces** | Request flow across services (spans, trace IDs) | Jaeger, Zipkin, AWS X-Ray, OpenTelemetry |

### What to Monitor in Production

| Category | Key Metrics |
|----------|-------------|
| Infrastructure | CPU, memory, disk I/O, network throughput |
| Application | Request rate, error rate, latency (p50/p95/p99) |
| Business | Signups, orders, revenue, conversion rate |
| Dependencies | DB query time, cache hit ratio, external API latency |
| Availability | Uptime %, health check status, pod restart count |

### Observability Tools Comparison

| Tool | Type | Strength | Weakness |
|------|------|----------|----------|
| Prometheus + Grafana | Metrics + Viz | Open-source, K8s-native, powerful queries | No built-in log or trace support |
| ELK Stack | Logs | Full-text search, powerful aggregations | Resource-heavy, complex to operate |
| Jaeger | Traces | Open-source, OpenTelemetry compatible | Needs separate metrics/logging |
| Datadog | All-in-one | SaaS, easy setup, all three pillars | Expensive at scale |
| Loki + Grafana | Logs | Lightweight, label-based, pairs with Prometheus | Less powerful querying than ELK |

### Linux Performance Observability Tools

| Tool | What It Reports |
|------|----------------|
| `vmstat` | Processes, memory, paging, block IO, CPU activity |
| `iostat` | CPU and input/output statistics |
| `netstat` | IP, TCP, UDP, ICMP protocol statistics |
| `lsof` | Open files of the current system |
| `pidstat` | Per-process CPU, memory, IO, task switching |
| `top` / `htop` | Real-time process resource usage |
| `strace` | System calls and signals |
| `perf` | CPU profiling, hardware counters |

## 9. Production Web Application Components

| # | Component | Role |
|---|-----------|------|
| 1 | **DNS** | Resolves domain → IP; first step in every request |
| 2 | **CDN** | Serves static assets from edge locations close to users (Fastly, CloudFront) |
| 3 | **Load Balancer** | Distributes traffic across app servers (Nginx, HAProxy, ALB) |
| 4 | **API Gateway** | Request routing, auth, rate limiting, protocol translation |
| 5 | **Application Servers** | Run business logic; stateless, horizontally scalable |
| 6 | **Database** | Persistent data storage (PostgreSQL, MySQL, DynamoDB) |
| 7 | **Cache** | Hot data in memory to reduce DB load (Redis, Memcached) |
| 8 | **Message Queue** | Async processing, decouple producers/consumers (Kafka, RabbitMQ) |
| 9 | **Full-Text Search** | Search functionality (Elasticsearch, Apache Solr) |
| 10 | **Monitoring & Alerting** | Logs, metrics, traces + alerting (Prometheus, Grafana, Sentry, PagerDuty) |

```
User → DNS → CDN → Load Balancer → API Gateway → App Servers
                                                      │
                                    ┌─────────┬────────┼────────┬──────────┐
                                    ▼         ▼        ▼        ▼          ▼
                                  Cache      DB    Msg Queue  Search   Monitoring
```

## 10. Configuration & Secrets Management

### Configuration Management

| Approach | When | Tools |
|----------|------|-------|
| Environment variables | Simple key-value config | dotenv, K8s ConfigMaps |
| Config files | Structured config (YAML/JSON) | Helm values, Spring profiles |
| Remote config service | Dynamic config, feature flags | Consul, etcd, AWS AppConfig, LaunchDarkly |

**Principles**: separate config from code, never hard-code env-specific values, use different configs per environment (dev/staging/prod).

### Secrets Management

| Practice | Description |
|----------|-------------|
| Never in source code | No passwords, API keys, or tokens in Git |
| Encryption at rest | Secrets encrypted in storage |
| Encryption in transit | SSL/TLS for all secret transmission |
| Rotation | Automate credential rotation |
| Least privilege | Grant minimal necessary access (RBAC) |
| Audit logging | Track who accessed which secret and when |

**Tools**: HashiCorp Vault, AWS Secrets Manager, AWS SSM Parameter Store, Azure Key Vault, Kubernetes Secrets (base64 only — pair with Sealed Secrets or external operator).

**Key management**: Separate roles — password applicant, password manager, and auditor each hold one piece of the key. All three required to unlock.

## 11. Design Patterns for Microservices

| Pattern | Problem It Solves | How It Works |
|---------|-------------------|-------------|
| **API Gateway** | Clients calling many services | Single entry point; routes, aggregates, and transforms requests |
| **Service Mesh** | Secure service-to-service communication | Sidecar proxies handle mTLS, retries, observability (Istio, Linkerd) |
| **Sidecar** | Cross-cutting concerns (logging, auth) | Helper container in the same Pod as the main container |
| **Circuit Breaker** | Cascading failures | Stop calling a failing service; fail fast, recover after timeout |
| **Bulkhead** | One failure taking down everything | Isolate resources per service/consumer (thread pools, connection pools) |
| **Saga** | Distributed transactions | Chain of local transactions with compensating actions on failure |
| **CQRS** | Read/write at different scales | Separate command (write) and query (read) models |
| **Event Sourcing** | Audit trail, temporal queries | Store state as sequence of events rather than current snapshot |
| **Strangler Fig** | Migrating from monolith | Gradually replace monolith capabilities with microservices behind a facade |
| **Backend for Frontend (BFF)** | Multiple client types with different needs | Each frontend gets its own mini backend for orchestration |

### Circuit Breaker State Machine

```
         success
  ┌──────────────┐
  ▼              │
CLOSED ──failures exceed threshold──▶ OPEN
                                       │
                                 timeout expires
                                       │
                                       ▼
                                  HALF-OPEN
                                   │      │
                              success    failure
                                │          │
                                ▼          ▼
                             CLOSED      OPEN
```
