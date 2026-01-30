# Repository Structure

## Monorepo Organization

```
platform/
├── apps/
│   ├── api/              # Core API service
│   ├── admin/            # Admin dashboard
│   ├── web/              # Main web application
│   └── mobile/           # Mobile app code
├── libs/
│   ├── core/             # Core utilities and models
│   ├── gis/              # GIS processing utilities
│   ├── ui/               # Shared UI components
│   └── auth/             # Authentication libraries
├── services/
│   ├── edtech/           # EdTech microservices
│   ├── gis/              # GIS microservices
│   └── analytics/        # Analytics microservices
├── scripts/              # Development and build scripts
├── docs/                 # Documentation
└── infrastructure/       # Kubernetes and infrastructure
```

## Service Structure

Each service follows a consistent structure:

```
service/
├── src/
│   ├── controllers/      # Request handlers
│   ├── models/           # Data models
│   ├── services/         # Business logic
│   ├── repositories/     # Data access
│   ├── utils/            # Utilities
│   └── index.ts          # Entry point
├── tests/
│   ├── unit/             # Unit tests
│   ├── integration/      # Integration tests
│   └── e2e/              # End-to-end tests
├── Dockerfile            # Container definition
├── tsconfig.json         # TypeScript config
├── package.json          # Dependencies
└── README.md             # Service documentation
```

## Environment Configuration

### Development (.env.development)

```bash
NODE_ENV=development
LOG_LEVEL=debug
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=platform_dev
```

### Testing (.env.test)

```bash
NODE_ENV=test
LOG_LEVEL=error
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=platform_test
```

### Production (.env.production)

```bash
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
