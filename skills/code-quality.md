# Code Quality Standards

## DRY Principle

Extract maximum leverage from every line of code:

- **Internal Libraries**: Package reusable logic into versioned modules importable across services
- **Bounded Contexts**: Define clear domain boundaries to prevent leaky abstractions
- **Contract Testing**: Ensure interfaces between components remain stable as implementations evolve
- **Code Generation**: Use templating/scaffolding for repetitive structures (routes, models, CRUD)
- **Automation Rule**: If done manually twice, automate the third time

## SOLID Principles

Engineer for resilience and maintainability:

- **Single Responsibility**: Services do one thing extremely well
- **Open/Closed**: Add capabilities through extension, not modification
- **Liskov Substitution**: Components hot-swappable with zero downstream impact
- **Interface Segregation**: Keep interfaces thin and focused—5 small interfaces > 1 large one
- **Dependency Injection**: Wire dependencies at runtime for maximum flexibility and testability

## Code Style Standards

- Format on save enabled
- ESLint/Prettier configuration enforced
- TypeScript strict mode required
- No `any` types without explicit justification
- Consistent naming conventions: camelCase (variables), PascalCase (components), SCREAMING_SNAKE_CASE (constants)
- Maximum line length: 100 characters

## Refactoring Strategy

- **Boy Scout Rule**: Leave code better than found
- **Tech Debt Backlog**: Maintain prioritized list of improvement opportunities
- **Refactoring Cadence**: Dedicate 20% of capacity to system improvement
- **Metric-Driven**: Focus on high-maintenance areas (error rates, time to deploy)

## Performance Optimization

- **Database Efficiency**: Profile slow queries, connection pooling, strategic indexing
- **Frontend Performance**: Bundle size <200KB initial load, TTI <3s on mid-tier mobile
- **Caching Layer**: Redis for frequently accessed data
- **Performance Budget**: Set clear limits for key metrics
