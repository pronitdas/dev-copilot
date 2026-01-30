# Development Workflow

## Git Workflow

Follow trunk-based development:

1. Pull latest changes from `main`
2. Create a feature branch `feature/description`
3. Make changes with regular commits
4. Push branch and create a PR
5. Address review comments
6. Squash and merge to `main`
7. Delete feature branch

## Commit Messages

Follow conventional commits structure:

```
type(scope): description

body

footer
```

### Types

- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Formatting, missing semicolons, etc.
- `refactor`: Code restructuring without behavior change
- `perf`: Performance improvements
- `test`: Adding or modifying tests
- `chore`: Maintenance tasks

### Examples

```
feat(map): add vector tile support
fix(auth): prevent token reuse after password change
docs: update API documentation
```

## Pull Request Requirements

### PR Template

```markdown
## Description
[Description of changes]

## Type of change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## How has this been tested?
- [ ] Unit tests
- [ ] Integration tests
- [ ] Manual tests

## Checklist
- [ ] Code follows style guidelines
- [ ] Tests added proving fix/feature works
- [ ] Documentation updated
- [ ] No performance impacts

## Related Issues
Fixes #123
```

## Branch Protection

- Require reviews from code owners
- Require passing CI checks
- Require linear history (squash merge)
- Require signed commits for sensitive changes

## Pre-commit Hooks

Use Husky and lint-staged:

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "pre-push": "npm test"
    }
  },
  "lint-staged": {
    "*.{js,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.json": ["prettier --write"]
  }
}
```
