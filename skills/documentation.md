# Documentation Standards

## Core Requirement

**Undocumented service = unsupported service.**

- Every service must have comprehensive documentation
- APIs require clear examples and error scenarios
- Architecture decisions must be recorded
- Documentation is a deliverable, not an add-on

## Documentation Types

### API Documentation

- Auto-generated from schemas
- Clear examples for all endpoints
- Error response documentation
- Authentication requirements
- Rate limiting info

### Architecture Decision Records (ADRs)

Document major decisions with:

1. Context and problem statement
2. Decision drivers and constraints
3. Options considered
4. Decision outcome and consequences
5. Validation criteria

### Runbooks

For operational scenarios:

- Deployment procedures
- Incident response steps
- Rollback procedures
- Monitoring and alerting
- Common troubleshooting steps

### Code Documentation

- Complex logic requires comments
- Public APIs must have docstrings
- Avoid documenting obvious code
- Keep comments up to date with code

## Implementation

- Auto-generated API docs from schemas
- ADRs for all major architectural decisions
- Runbooks for operational scenarios
- Clear onboarding paths for new developers

## Enforcement

- Documentation coverage checks
- Broken doc links block merges
- Documentation freshness metrics
- Required reviewer for documentation changes
- Documentation reviewed as part of PR

## README Template

Every service must have:

```
# Service Name

## Description
Brief overview of what this service does.

## Quick Start
Steps to run locally.

## Architecture
High-level design and data flow.

## API
Links to API documentation.

## Configuration
Environment variables and secrets.

## Development
Setup instructions, testing, contributing.

## Deployment
CI/CD pipeline, deployment process.

## Monitoring
Health checks, metrics, alerting.

## Troubleshooting
Common issues and solutions.
```
