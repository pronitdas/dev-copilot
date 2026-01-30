# Engineering Fundamentals

High-leverage patterns for maximum output with minimal maintenance.

---

## Core Principles

### DRY — Extract Maximum Leverage

| Pattern | Implementation |
|---------|----------------|
| Internal Libraries | Versioned modules imported across services |
| Bounded Contexts | Clear domain boundaries prevent leaks |
| Contract Testing | Stable interfaces as implementations evolve |
| Code Generation | Templates for routes, models, CRUD |
| Automation | Manual twice → automate the third time |

### SOLID — Engineer for Resilience

| Principle | Application |
|-----------|-------------|
| Single Responsibility | Services do one thing extremely well |
| Open/Closed | Extend through composition, not modification |
| Liskov Substitution | Hot-swappable components, zero downstream impact |
| Interface Segregation | 5 small interfaces > 1 large one |
| Dependency Injection | Runtime wiring for flexibility and testability |

### 12-Factor — Cloud-Native By Default

| Factor | Implementation | Benefit |
|--------|----------------|---------|
| Codebase | One repo per service | No merge conflicts |
| Dependencies | Explicit declarations | Reproducible builds |
| Config | Environment variables | No config in code |
| Backing Services | Attached resources | Seamless environments |
| Build/Release/Run | CI/CD with immutable artifacts | Predictable deployments |
| Processes | Stateless + external persistence | Horizontal scaling |
| Port Binding | Self-contained HTTP services | Clear boundaries |
| Concurrency | Scale via process model | Predictable resources |
| Disposability | Fast startup, graceful shutdown | Zero-downtime deployments |
| Dev/Prod Parity | Container-based development | Works everywhere |
| Logs | Structured JSON to stdout | Searchable, aggregatable |
| Admin Tasks | Packaged with application code | Reproducible operations |

---

## Infrastructure Standards

### Kubernetes-Ready Architecture

- **Stateless Core** — State in Postgres, Redis, S3
- **Health Checks** — `/health` and `/ready` endpoints
- **Configuration** — ConfigMaps for app config, Secrets for credentials
- **Resource Limits** — Explicit CPU/memory requirements
- **Graceful Shutdown** — Catch SIGTERM, drain connections

### Observability Stack

| Pillar | Tool | Purpose |
|--------|------|---------|
| Metrics | Prometheus | Time-series monitoring |
| Logs | Structured JSON | Searchable logs |
| Traces | OpenTelemetry | Request flow across services |
| Alerts | SLO-based | Alert on burn rate, not raw metrics |

**Four Golden Signals:**
- Latency — How long to respond?
- Traffic — Requests/second?
- Errors — Failure percentage?
- Saturation — Resource utilization?

---

## Security & Compliance

### HIPAA Requirements

| Requirement | Implementation |
|-------------|----------------|
| Encryption | TLS 1.3+ in transit, AES-256 at rest |
| Access Control | RBAC with principle of least privilege |
| Audit Logging | Record all PHI access (who/what/when/why) |
| Data Segmentation | Isolate PHI in dedicated storage |
| Backup Strategy | Regular restore testing |
| BAAs | Business Associate Agreements with vendors |
| Incident Response | Documented breach procedures |

### PCI-DSS Requirements

| Requirement | Implementation |
|-------------|----------------|
| Network Segmentation | Isolate payment processing |
| Sensitive Data | Tokenization over storage |
| Vulnerability Scanning | Automated regular scanning |
| Penetration Testing | Annual third-party assessment |
| Patch Management | Clear security update process |
| Minimal Surface Area | Remove unnecessary services |

---

## Development Workflow

### Version Control

- **Trunk-Based** — Short-lived branches, merge daily
- **PR Templates** — Standardized format with risk assessment
- **Semantic Versioning** — Strict semver for all libraries
- **Commit Format** — `type(scope): message` (e.g., `feat(auth): add MFA`)
- **Branch Protection** — Reviews + passing CI required

### Testing Strategy

| Level | Purpose | Tools |
|-------|---------|-------|
| Unit | Business logic in isolation | Jest |
| Integration | Service interactions | Supertest |
| Contract | API compatibility | Pact |
| Performance | Benchmarks with pass/fail | k6 |
| Chaos | Resilience under disruption | Chaos Mesh |

### CI/CD Requirements

| Metric | Target |
|--------|--------|
| Feedback cycle | <10 minutes |
| Deployment | Fully automated, progressive rollout |
| Feature Flags | Decouple deployment from release |
| Artifact Promotion | Build once, deploy many |
| Ephemeral Environments | On-demand test environments |

---

## Technical Debt Management

| Strategy | Description |
|----------|-------------|
| Boy Scout Rule | Leave code better than found |
| Tech Debt Backlog | Prioritized improvement list |
| Refactoring Cadence | 20% capacity to system improvement |
| Deprecation Policy | Clear sunset timelines for old APIs |
| Metric-Driven | Focus on high-maintenance areas |

---

## Performance Optimization

### Database

| Optimization | Description |
|--------------|-------------|
| Query Optimization | Profile and tune slow queries |
| Connection Pooling | Reuse connections |
| Index Strategy | Create based on access patterns |
| Caching Layer | Redis for frequently accessed data |
| Partitioning | Horizontal splitting for large tables |

### Frontend

| Metric | Target |
|--------|--------|
| Bundle Size | <200KB initial JS |
| Time to Interactive | <3s on mid-tier mobile |
| Lazy Loading | Split code by route |
| CDN | Edge caching for static assets |

---

## Key Metrics

| Metric | Target | Why |
|--------|--------|-----|
| Deployment Frequency | Daily | Velocity indicator |
| Lead Time for Changes | <24h | Organizational agility |
| Change Failure Rate | <15% | Quality indicator |
| Time to Recovery | <15min | Resilience |
| P95 Latency | <300ms | User experience |
| Test Coverage | >85% | Change confidence |
| CPU/Memory Utilization | <70% | Headroom for spikes |
| Error Budget Consumption | <80% | Innovation vs stability |

---

## Architecture Decision Records

ADRs document:
1. Context and problem statement
2. Decision drivers and constraints
3. Options considered
4. Decision outcome and consequences
5. Validation criteria

---

## Final Principles

1. **Optimize for Deletion** — Best code is replaceable code
2. **Design for Operations** — Build with monitoring in mind
3. **Data > Opinions** — Metrics over hunches
4. **Automate Toil** — Script repetitive tasks
5. **Embrace Simplicity** — Complex solutions compound costs

> Speed of iteration beats perfection. Build modular systems that evolve component by component.
