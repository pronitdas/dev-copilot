# Component Ownership

## Core Principle

**One team, one component.**

Every component has clear ownership with one team responsible for:

- Performance
- Reliability
- Documentation
- Evolution

## Ownership Model

### Public Interface

```
┌─────────────────────────────────────────┐
│                                         │
│  Component                              │
│  ┌─────────────────────────────────┐    │
│  │                                 │    │
│  │  Public Interface ← Team A owns │    │
│  │                                 │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │                                 │    │
│  │  Internal Implementation        │    │
│  │  ← Team A has full autonomy     │    │
│  │                                 │    │
│  └─────────────────────────────────┘    │
│                                         │
└─────────────────────────────────────────┘
```

## Rules of Engagement

1. **Interface Stability**: Public interfaces must be versioned and backwards compatible
2. **Implementation Freedom**: Teams can refactor internal implementations without approval
3. **SLA Ownership**: Teams define and meet their own SLAs
4. **Cross-cutting Changes**: Require RFC and explicit approval from all affected owners

## Ownership Responsibilities

### Code Owners

- Review all changes to owned code
- Respond to incidents within SLA
- Maintain documentation
- Plan and execute improvements
- Communicate changes to consumers

### When You Break It, You Own It

- Ownership follows contribution
- Take responsibility for your changes
- Fix broken tests before adding features
- Understand before modifying

## Component Registry

All components registered in `registry.json` with:

- Owning team and contacts
- Primary and secondary contacts
- SLA commitments
- Dependency graph

## Enforcement

- CODEOWNERS files map services to teams
- Automated assignment of issues based on blame
- Required reviews from service owners
- Post-incident reviews assign clear action items
- Breaking changes require owner approval
