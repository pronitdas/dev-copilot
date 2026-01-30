# Development Guide

Daily workflow for productive development.

---

## 1. Repository Structure

```
platform/
├── apps/
│   ├── api/           # Core API service
│   ├── admin/         # Admin dashboard
│   ├── web/           # Main web application
│   └── mobile/        # Mobile app
├── libs/
│   ├── core/          # Core utilities and models
│   ├── gis/           # GIS processing utilities
│   ├── ui/            # Shared UI components
│   └── auth/          # Authentication libraries
├── services/
│   ├── edtech/        # EdTech microservices
│   ├── gis/           # GIS microservices
│   └── analytics/     # Analytics microservices
├── scripts/           # Development/build scripts
├── docs/              # Documentation
└── infrastructure/    # Kubernetes configs
```

### Service Template

```
service/
├── src/
│   ├── controllers/   # Request handlers
│   ├── models/        # Data models
│   ├── services/      # Business logic
│   ├── repositories/  # Data access
│   ├── utils/         # Utilities
│   └── index.ts       # Entry point
├── tests/
│   ├── unit/          # Unit tests
│   ├── integration/   # Integration tests
│   └── e2e/           # End-to-end tests
├── Dockerfile
├── tsconfig.json
├── package.json
└── README.md
```

---

## 2. VSCode Setup

### Required Extensions

```bash
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension ms-azuretools.vscode-docker
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
code --install-extension visualstudioexptteam.vscodeintellicode
code --install-extension eamodio.gitlens
code --install-extension gruntfuggly.todo-tree
```

### Workspace Settings

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": { "source.fixAll.eslint": true },
  "editor.rulers": [100],
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.preferences.importModuleSpecifier": "relative",
  "eslint.validate": ["typescript", "typescriptreact"]
}
```

### Keyboard Shortcuts

| Function | macOS | Windows |
|----------|-------|---------|
| Format Document | `Cmd+Opt+F` | `Shift+Alt+F` |
| Go to Definition | `F12` | `F12` |
| Find References | `Cmd+Shift+F12` | `Shift+F12` |
| Rename Symbol | `F2` | `F2` |
| Quick Fix | `Cmd+.` | `Ctrl+.` |
| Toggle Terminal | `Cmd+`` | `Ctrl+``` |
| Search Files | `Cmd+P` | `Ctrl+P` |
| Run Tests | `Cmd+Shift+T` | `Ctrl+Shift+T` |

---

## 3. Environment Configuration

### Dotenv Files

| File | Purpose |
|------|---------|
| `.env.development` | Local development settings |
| `.env.test` | Test environment |
| `.env.production` | Production (values injected) |

**Example (.env.development):**
```bash
NODE_ENV=development
LOG_LEVEL=debug
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=platform_dev
```

---

## 4. Git Workflow

### Trunk-Based Development

1. Pull latest from `main`
2. Create branch: `feature/description`
3. Commit changes regularly
4. Push and create PR
5. Address review comments
6. Squash and merge
7. Delete feature branch

### Commit Format

```
type(scope): description

body (optional)

footer (breaking changes, issues)
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`

**Examples:**
```
feat(map): add vector tile support
fix(auth): prevent token reuse after password change
```

---

## 5. Pre-commit Hooks

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "pre-push": "npm test"
    }
  },
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.json": ["prettier --write"]
  }
}
```

---

## 6. Testing Practices

### Unit Test Pattern

```typescript
describe('MarkerService', () => {
  let service: MarkerService;
  let repository: jest.Mocked<MarkerRepository>;

  beforeEach(() => {
    repository = new MarkerRepository() as jest.Mocked<MarkerRepository>;
    service = new MarkerService(repository);
  });

  it('should create a marker with valid data', async () => {
    const markerData = {
      name: 'Test Marker',
      position: { latitude: 10, longitude: 20 }
    };

    repository.create.mockResolvedValue({ id: '123', ...markerData });

    const result = await service.createMarker(markerData);

    expect(repository.create).toHaveBeenCalledWith(markerData);
    expect(result).toHaveProperty('id', '123');
  });
});
```

### Integration Test Pattern

```typescript
describe('MarkerRepository', () => {
  beforeAll(async () => {
    db = await createTestDatabase();
    repository = new MarkerRepository(db);
  });

  it('should store and retrieve markers', async () => {
    const marker = await repository.create({
      name: 'Test Marker',
      position: { latitude: 10, longitude: 20 }
    });

    expect(marker).toHaveProperty('id');

    const retrieved = await repository.findById(marker.id);
    expect(retrieved).toMatchObject({
      name: 'Test Marker',
      position: { latitude: 10, longitude: 20 }
    });
  });
});
```

### Testing Tools

| Type | Tool | Purpose |
|------|------|---------|
| Unit | Jest | Business logic |
| API | Supertest | HTTP testing |
| E2E | Cypress | Browser testing |
| Browser | Playwright | Cross-browser |

---

## 7. Task Tracking

### Issue Template

```markdown
## Description
[Clear description]

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Technical Notes
[Implementation details]

## Dependencies
- #123 (other issue)
```

### PR Template

```markdown
## Description
[Changes description]

## Type
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation

## Testing
- [ ] Unit tests
- [ ] Integration tests
- [ ] Manual tests

## Checklist
- [ ] Follows style guidelines
- [ ] Tests added
- [ ] Documentation updated
- [ ] No performance impact
```

---

## 8. Debugging

### Enable Debug Logging

```bash
LOG_LEVEL=debug
```

### Filter Logs by Request ID

```bash
grep "request_id=req_123456" logs/app.log
```

### LLM Request Logging

```typescript
logger.info('LLM Request', {
  request_id: requestId,
  model: 'gpt-4',
  latency_ms: endTime - startTime
});
```

---

## Quick Reference

| Task | Command/Action |
|------|----------------|
| Format code | `npm run format` |
| Run tests | `npm test` |
| Lint | `npm run lint` |
| Build | `npm run build` |
| Start dev | `npm run dev` |

**See Also:**
- [Design Principles](./design-principles.md) — Architecture decisions
- [Fundamentals](./fundamentals.md) — Engineering standards
- [Glossary](./glossary.md) — Technology definitions
- [Local Setup](./local-setup.md) — Environment setup
