# Rules of Engagement

Non-negotiable standards for all developers. These are requirements, not suggestions.

---

## 1. Precision

**Schema contracts are law, not guidelines.**

| Requirement | Implementation |
|-------------|----------------|
| API validation | All requests/responses validate against schema |
| No interpretation | Fields either meet contract or fail |
| Versioning | Schema changes require explicit migration paths |
| Undefined behavior | Log as critical error, not a feature |

**Enforcement:**
- PR checks verify schema compliance
- Schema regression tests on every commit
- Production violations trigger alerts

---

## 2. Security

**Security is a requirement, not an afterthought.**

| Requirement | Implementation |
|-------------|----------------|
| Input validation | All inputs validated and sanitized |
| Secrets management | No secrets in code, configs, or logs |
| Authentication | 2FA required for all systems |
| Authorization | Zero-trust between all services |

**Enforcement:**
- Static analysis for leaked secrets
- Security scans on all dependencies
- Automated penetration testing
- Failed audits block releases

---

## 3. Packaging

**One container = one purpose.**

| Requirement | Implementation |
|-------------|----------------|
| Single responsibility | Every service does one thing well |
| Clear boundaries | No monolithic containers |
| Explicit dependencies | Documented and versioned |
| Minimal images | Multi-stage builds, distroless bases |

**Enforcement:**
- Container size limits
- Manifest validation
- Dependency scanning
- Container immutability in runtime

---

## 4. Ownership

**You break it, you own it.**

| Requirement | Implementation |
|-------------|----------------|
| Responsibility | Ownership follows contribution |
| Understanding | Understand before modifying |
| Tests | Fix broken tests before adding features |

**Enforcement:**
- CODEOWNERS files map services to teams
- Required reviews from service owners
- Post-incident reviews with action items
- On-call rotation follows ownership

---

## 5. Testing

**PRs without tests = PRs denied.**

| Requirement | Implementation |
|-------------|----------------|
| Test coverage | No untested code reaches production |
| Error conditions | Tests cover happy path AND failures |
| Integration | As important as unit tests |
| Performance | Required for critical paths |

**Enforcement:**
- Automated coverage thresholds
- Chaos testing for resiliency
- Failed tests block deployment
- Metrics tracked per developer

---

## 6. Observability

**Always log with trace IDs.**

| Requirement | Implementation |
|-------------|----------------|
| Trace context | Every request has unique trace ID |
| Structured logging | No unstructured text, machine-parseable |
| Propagation | Log context passed through all calls |

**Enforcement:**
- Log validation in CI
- Alerting on trace discontinuities
- Sampling for all production traffic
- Retention policies enforced

---

## 7. Documentation

**Undocumented service = unsupported service.**

| Requirement | Implementation |
|-------------|----------------|
| Service docs | Every service has comprehensive docs |
| API examples | Clear examples and error scenarios |
| Architecture | ADRs for all major decisions |
| Deliverable | Documentation is a deliverable, not an add-on |

**Enforcement:**
- Documentation coverage checks
- Broken doc links block merges
- Freshness metrics tracked
- Required reviewer for doc changes

---

## Violation Consequences

| Level | Consequence |
|-------|-------------|
| 1st Violation | Warning with required remediation |
| 2nd Violation | Freeze on new features until practices improve |
| Repeated | Reassignment to less critical systems |
| Systemic | Removal from project |

---

## Why These Rules Matter

These rules exist to:

- **Reduce cognitive load** — Consistent patterns make systems predictable
- **Improve resilience** — Proper testing and validation prevent outages
- **Enable scaling** — Clear boundaries and ownership allow parallel work
- **Maintain velocity** — Upfront discipline prevents downstream slowdowns
- **Protect users** — Security and correctness aren't negotiable

---

## Quick Reference

| Rule | Key Principle |
|------|---------------|
| Precision | Schema first |
| Security | Zero trust |
| Packaging | Single purpose |
| Ownership | You break it, you own it |
| Testing | Test > guess |
| Observability | Trace everything |
| Documentation | Document or die |

> The easiest person to debug code for is your future self. Write as if the next person to maintain your code is a psychopath who knows where you live.
