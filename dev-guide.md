# Development Guide

This document covers the daily development workflow for our platform. It's designed to maximize productivity while maintaining quality.

## Repo Structure

Our codebase is organized into a monorepo structure with clear separation:

```
platform/
├── apps/
│   ├── api/                # Core API service
│   ├── admin/              # Admin dashboard
│   ├── web/                # Main web application
│   └── mobile/             # Mobile app code
├── libs/
│   ├── core/               # Core utilities and models
│   ├── gis/                # GIS processing utilities 
│   ├── ui/                 # Shared UI components
│   └── auth/               # Authentication libraries
├── services/
│   ├── edtech/             # EdTech microservices
│   ├── gis/                # GIS microservices
│   └── analytics/          # Analytics microservices
├── scripts/                # Development and build scripts
├── docs/                   # Documentation
└── infrastructure/         # Kubernetes and infrastructure
```

### Service Boundaries

Each service follows a consistent structure:

```
service/
├── src/
│   ├── controllers/        # Request handlers
│   ├── models/             # Data models
│   ├── services/           # Business logic
│   ├── repositories/       # Data access
│   ├── utils/              # Utilities
│   └── index.ts            # Entry point
├── tests/
│   ├── unit/               # Unit tests
│   ├── integration/        # Integration tests
│   └── e2e/                # End-to-end tests
├── Dockerfile              # Container definition
├── tsconfig.json           # TypeScript config
├── package.json            # Dependencies
└── README.md               # Service documentation
```

## VSCode Setup

For maximum productivity, we use a standardized VSCode setup:

### Required Extensions

Install these extensions for optimal development:

```
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension ms-azuretools.vscode-docker
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
code --install-extension ms-vscode.vscode-typescript-tslint-plugin
code --install-extension visualstudioexptteam.vscodeintellicode
code --install-extension eamodio.gitlens
code --install-extension gruntfuggly.todo-tree
```

### Workspace Settings

Create a `.vscode/settings.json` file with:

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "editor.rulers": [100],
  "files.exclude": {
    "**/.git": true,
    "**/.DS_Store": true,
    "**/node_modules": true,
    "**/dist": true,
    "**/*.js.map": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.preferences.importModuleSpecifier": "relative",
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact"
  ],
  "todo-tree.general.tags": [
    "TODO",
    "FIXME",
    "HACK",
    "BUG",
    "PERFORMANCE"
  ],
  "todo-tree.highlights.defaultHighlight": {
    "icon": "alert",
    "type": "text",
    "foreground": "#FF0000",
    "background": "#FFFF00",
    "opacity": 50,
    "iconColour": "#FF0000"
  }
}
```

### Recommended Keyboard Shortcuts

For maximum productivity:

| Function | Windows | macOS |
|----------|---------|-------|
| Format Document | Shift+Alt+F | Shift+Option+F |
| Go to Definition | F12 | F12 |
| Find References | Shift+F12 | Shift+F12 |
| Rename Symbol | F2 | F2 |
| Quick Fix | Ctrl+. | Cmd+. |
| Toggle Terminal | Ctrl+` | Cmd+` |
| Search Files | Ctrl+P | Cmd+P |
| Run Tests | Ctrl+Shift+T | Cmd+Shift+T |

## Dotenv Setup

Every service uses `.env` files for configuration. Never commit these files to git.

**Development Environment:**
```
# .env.development
NODE_ENV=development
LOG_LEVEL=debug
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=platform_dev
```

**Testing Environment:**
```
# .env.test
NODE_ENV=test
LOG_LEVEL=error
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=platform_test
```

**Production Environment:**
```
# .env.production
# This file should only contain keys, not values
# Values are injected by the deployment system
NODE_ENV=production
LOG_LEVEL=
POSTGRES_HOST=
POSTGRES_PORT=
POSTGRES_USER=
POSTGRES_PASSWORD=
POSTGRES_DB=
```

## Git Workflow

We follow a trunk-based development workflow:

1. Pull latest changes from `main`
2. Create a feature branch `feature/description`
3. Make changes with regular commits
4. Push branch and create a PR
5. Address review comments
6. Squash and merge to `main`
7. Delete feature branch

### Commit Messages

Follow conventional commits structure:

```
type(scope): description

body

footer
```

Where:
- `type` is one of: feat, fix, docs, style, refactor, perf, test, chore
- `scope` is the module affected
- `description` is a short summary
- `body` contains detailed explanation (optional)
- `footer` contains breaking changes or issue references (optional)

Examples:
```
feat(map): add vector tile support

Added MapboxVectorTile support to GIS renderer for better performance.

Closes #123
```

```
fix(auth): prevent token reuse after password change

BREAKING CHANGE: all existing tokens are invalidated on password change
```

## Pre-commit Hooks

We use Husky and lint-staged to enforce quality:

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "pre-push": "npm test"
    }
  },
  "lint-staged": {
    "*.{js,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.json": [
      "prettier --write"
    ]
  }
}
```

## Testing Practices

All code must have tests. We use:

- **Jest** for unit testing
- **Supertest** for API testing
- **Cypress** for E2E testing
- **Playwright** for browser testing

### Unit Tests

Follow this pattern for unit tests:

```typescript
// src/services/__tests__/marker.service.test.ts
import { MarkerService } from '../marker.service';
import { MarkerRepository } from '../../repositories/marker.repository';

// Mock dependencies
jest.mock('../../repositories/marker.repository');

describe('MarkerService', () => {
  let service: MarkerService;
  let repository: jest.Mocked<MarkerRepository>;
  
  beforeEach(() => {
    repository = new MarkerRepository() as jest.Mocked<MarkerRepository>;
    service = new MarkerService(repository);
  });
  
  afterEach(() => {
    jest.resetAllMocks();
  });
  
  describe('createMarker', () => {
    it('should create a marker with valid data', async () => {
      // Arrange
      const markerData = {
        name: 'Test Marker',
        position: { latitude: 10, longitude: 20 }
      };
      
      repository.create.mockResolvedValue({
        id: '123',
        ...markerData,
        createdAt: new Date(),
        updatedAt: new Date()
      });
      
      // Act
      const result = await service.createMarker(markerData);
      
      // Assert
      expect(repository.create).toHaveBeenCalledWith(markerData);
      expect(result).toHaveProperty('id', '123');
      expect(result).toHaveProperty('name', 'Test Marker');
    });
    
    it('should throw error when position is invalid', async () => {
      // Arrange
      const markerData = {
        name: 'Test Marker',
        position: { latitude: 100, longitude: 20 } // Invalid latitude
      };
      
      // Act & Assert
      await expect(service.createMarker(markerData))
        .rejects
        .toThrow('Invalid position');
      
      expect(repository.create).not.toHaveBeenCalled();
    });
  });
});
```

### Integration Tests

For integration tests with databases:

```typescript
// src/tests/integration/markers.test.ts
import { createTestDatabase, closeTestDatabase } from '../utils/test-db';
import { MarkerRepository } from '../../repositories/marker.repository';

describe('MarkerRepository', () => {
  let db: any;
  let repository: MarkerRepository;
  
  beforeAll(async () => {
    db = await createTestDatabase();
    repository = new MarkerRepository(db);
  });
  
  afterAll(async () => {
    await closeTestDatabase(db);
  });
  
  beforeEach(async () => {
    await db.query('TRUNCATE markers CASCADE');
  });
  
  it('should store and retrieve markers', async () => {
    // Create a marker
    const marker = await repository.create({
      name: 'Test Marker',
      position: { latitude: 10, longitude: 20 }
    });
    
    expect(marker).toHaveProperty('id');
    
    // Retrieve the marker
    const retrieved = await repository.findById(marker.id);
    
    expect(retrieved).toMatchObject({
      name: 'Test Marker',
      position: { latitude: 10, longitude: 20 }
    });
  });
});
```

## Task Tracking

We use GitHub Projects for task tracking:

### Issue Structure

Create issues with the following template:

```markdown
## Description
[Clear description of the task]

## Acceptance Criteria
- [ ] [Criterion 1]
- [ ] [Criterion 2]
- [ ] [Criterion 3]

## Technical Notes
[Any implementation details, architecture decisions, etc.]

## Dependencies
- #123 (other issue)
- #456 (other issue)
```

### Pull Request Template

```markdown
## Description
[Description of the changes]

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
- [ ] My code follows the style guidelines
- [ ] I have added tests that prove my fix/feature works
- [ ] I have updated documentation
- [ ] I have checked for performance impacts

## Related Issues
Fixes #123
```

## Debugging Tips

### LLM Logs

LLM requests/responses are logged in structured format:

```typescript
logger.info('LLM Request', {
  request_id: requestId,
  prompt: truncateForLogs(prompt, 500),
  model: 'gpt-4',
  temperature: 0.7,
  max_tokens: 1024
});

logger.info('LLM Response', {
  request_id: requestId,
  completion: truncateForLogs(completion, 500),
  tokens: {
    prompt: tokenCount.prompt,
    completion: tokenCount.completion,
    total: tokenCount.total
  },
  latency_ms: endTime - startTime
});
```

To debug:

1. Enable debug logging in your `.env`:
   ```
   LOG_LEVEL=debug
   ```

2. Filter logs by request ID:
   ```bash
   grep "request_id=req_123456" logs/app.log
   ```
