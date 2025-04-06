# Rules of Engagement

This document outlines the non-negotiable standards for all developers working on our platform. These rules aren't suggestions - they're requirements.

## 💡 Precision

**Schema contracts are law, not guidelines.**

- Every API request and response must validate against its schema
- No field should ever be "interpreted" - it either meets the contract or fails
- Changes to schemas require versioning and explicit migration paths
- Undefined behavior is a bug, not a feature

**Implementation:**
- Use `zod`, `joi`, or equivalent validation at API boundaries
- Implement runtime type checking in addition to TypeScript
- Write tests that explicitly verify schema adherence
- Log schema violations as critical errors

**Enforcement:**
- PR checks automatically verify schema compliance
- Schema regression tests run on every commit
- Schema violations in production trigger alerts

## 🔐 Security

**Security isn't an afterthought - it's a requirement.**

- All inputs must be validated and sanitized
- No secrets in code, configs, or logs
- All developers must use 2FA for everything
- Zero-trust between all services

**Implementation:**
- All environment variables pass through vault
- Credentials auto-rotate on schedule and after incidents
- Containers run as non-root users with minimal permissions
- All APIs use authentication and authorization

**Enforcement:**
- Static analysis checks for leaked secrets
- Security scans on all dependencies
- Automated penetration testing
- Failed security audits block releases

## 📦 Packaging

**One container = one purpose**

- Every service does one thing well
- No monolithic containers
- Clear responsibility boundaries
- Explicit dependencies

**Implementation:**
- Multi-stage builds for minimal containers
- Distroless base images where possible
- Dependency vendoring for reproducible builds
- No shell access in production containers

**Enforcement:**
- Container size limits enforced
- Manifest validation for each service
- Automated dependency scanning
- Container immutability in runtime

## 🧠 Mental Model

**You break it, you own it.**

- Ownership follows contribution
- Take responsibility for your changes
- Fix broken tests before adding features
- Understand before modifying

**Implementation:**
- CODEOWNERS files map services to teams
- Automated assignment of issues based on blame
- Required reviews from service owners
- Post-incident reviews assign clear action items

**Enforcement:**
- Breaking changes require owner approval
- On-call rotation follows ownership
- Performance impact monitoring per contributor
- Recognition for responsible ownership

## 🧪 Test > Guess

**PRs without tests = PRs denied**

- No untested code reaches production
- Tests must cover happy path AND error conditions
- Integration tests are as important as unit tests
- Performance tests are required for critical paths

**Implementation:**
- Minimum code coverage thresholds
- Chaos testing for resiliency
- Load testing for performance guarantees
- E2E tests for critical user journeys

**Enforcement:**
- Automated PR checks for test coverage
- Performance regression testing
- Failed tests block deployment
- Test metrics tracked per developer

## 📝 Logs & Trace

**Always log with trace IDs**

- Every request has a unique trace ID
- Log context must be propagated
- Structured logging only - no unstructured text
- Logs must be machine-parseable

**Implementation:**
- OpenTelemetry for distributed tracing
- Consistent log format across all services
- Correlation IDs passed through all service calls
- Log levels appropriate to context

**Enforcement:**
- Log validation in CI pipeline
- Alerting on trace discontinuities
- Trace sampling for all production traffic
- Log retention policies enforced

## 📚 Docs or Die

**Undocumented service = unsupported service**

- Every service must have comprehensive docs
- APIs require clear examples and error scenarios
- Architecture decisions must be recorded
- Documentation is a deliverable, not an add-on

**Implementation:**
- Auto-generated API docs from schemas
- Architecture Decision Records (ADRs) for all major decisions
- Runbooks for operational scenarios
- Clear onboarding paths for new developers

**Enforcement:**
- Documentation coverage checks
- Broken docs links block merges
- Documentation freshness metrics
- Required reviewer for documentation changes

## Violation Consequences

These rules aren't optional. Violations have concrete consequences:

1. **First Violation**: Warning and required remediation
2. **Second Violation**: Temporary freeze on new feature work until practices improve
3. **Repeated Violations**: Reassignment to less critical systems
4. **Systemic Violations**: Removal from project

## Why These Rules Matter

These rules exist for a reason. They:

- **Reduce cognitive load** - consistent patterns make systems predictable
- **Improve resilience** - proper testing and validation prevent outages
- **Enable scaling** - clear boundaries and ownership allow parallel work
- **Maintain velocity** - upfront discipline prevents downstream slowdowns
- **Protect users** - security and correctness aren't negotiable

Remember: The easiest person to debug code for is your future self. Write as if the next person to maintain your code is a psychopath who knows where you live.
