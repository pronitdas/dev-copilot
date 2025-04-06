# Local Development Setup

This guide will get you from zero to a fully operational development environment in under 30 minutes. No fluff, just results.

## Prerequisites

### Essential Tools

Install these core dependencies first:

**For All Platforms:**
- Git 2.35+
- Docker Desktop 4.15+ (with Kubernetes enabled)
- Node.js 18+ (via nvm)
- Python 3.10+ (via pyenv)
- Visual Studio Code
- Postman (for API testing)

**Platform-Specific Requirements:**

#### Windows
```powershell
# Install Chocolatey if not already installed
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# Install core tools
choco install -y git nodejs-lts python kubernetes-cli docker-desktop
choco install -y visualstudiocode postman

# Enable Hyper-V (required for Docker)
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

#### macOS
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install core tools
brew install git node@18 python@3.10 kubectl
brew install --cask docker visual-studio-code postman

# Enable Kubernetes in Docker Desktop
# (Do this through the Docker Desktop UI: Settings -> Kubernetes -> Enable Kubernetes)
```

## Repository Setup

Clone all required repositories:

```bash
# Create development directory
mkdir -p ~/dev/platform
cd ~/dev/platform

# Clone repositories
git clone git@github.com:company/core-platform.git
git clone git@github.com:company/edtech-services.git
git clone git@github.com:company/gis-services.git
git clone git@github.com:company/frontend.git

# Initialize all submodules
find . -type d -depth 1 -exec bash -c "cd '{}' && git submodule update --init --recursive" \;
```

## Environment Configuration

Set up your local environment variables:

```bash
# Copy sample env files
find . -name ".env.example" -exec bash -c 'cp "$1" "${1%.example}"' _ {} \;

# Generate development keys (DO NOT USE THESE IN PRODUCTION)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout dev-cert-key.pem -out dev-cert.pem -subj "/CN=localhost"
```

Edit the following in your `.env` files:

**Core Platform (.env)**
```
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/core_platform_dev
REDIS_URL=redis://localhost:6379/0
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=dev-assets
JWT_SECRET=THIS_IS_NOT_A_PRODUCTION_SECRET_REPLACE_IT
```

**EdTech Services (.env)**
```
CORE_PLATFORM_URL=http://localhost:3000
OPENAI_API_KEY=YOUR_API_KEY_HERE # Required for LLM functionality
FFMPEG_THREADS=4 # Adjust based on your CPU
```

**GIS Services (.env)**
```
POSTGIS_URL=postgresql://postgres:postgres@localhost:5433/gis_dev
TITILER_ENDPOINT=http://localhost:8000
VECTOR_TILE_CACHE=./cache
MAX_ZOOM_LEVEL=16
DEFAULT_PROJECTION=EPSG:3857
```

## Database Setup

Launch the required databases:

```bash
# Start PostgreSQL, PostGIS, Redis, and MinIO
docker-compose -f docker/dev-infrastructure.yml up -d

# Initialize databases with schema
./scripts/init-databases.sh
```

## GIS Tilesets

Download sample tilesets for local development:

```bash
# Create GIS data directory
mkdir -p ~/dev/gis-data

# Download and extract sample data
curl -L https://github.com/company/sample-tilesets/releases/download/v1.0.0/sample-tilesets.tar.gz | tar -xz -C ~/dev/gis-data
```

Import tilesets into the local PostGIS instance:

```bash
# Import OSM sample data
cd ~/dev/platform/gis-services
./scripts/import-tilesets.sh ~/dev/gis-data
```

## Docker & Kubernetes

Configure local Kubernetes for development:

```bash
# Create development namespace
kubectl create namespace platform-dev

# Apply dev configs
kubectl apply -f k8s/dev-configs.yaml -n platform-dev

# Configure local Docker registry
kubectl apply -f k8s/local-registry.yaml
```

## Build and Run

Build and run all services:

```bash
# Build all Docker images
cd ~/dev/platform
./scripts/build-all.sh

# Start the development environment
./scripts/start-dev.sh
```

This script will:
1. Build all Docker images with dev tags
2. Deploy them to your local Kubernetes
3. Set up port forwarding for each service
4. Configure hot reloading for code changes

## Verifying Your Setup

Run the verification script to ensure everything is working:

```bash
./scripts/verify-setup.sh
```

The script will check:
- Database connectivity
- Redis connection
- S3/MinIO access
- Kubernetes pod health
- API endpoints
- GIS tile server functionality

If successful, you should see output like:
```
✓ PostgreSQL connection successful
✓ Redis connection successful
✓ MinIO connection successful
✓ All Kubernetes pods are running
✓ Core API endpoint responding
✓ EdTech API endpoint responding
✓ GIS API endpoint responding
✓ Tile server functioning correctly
```

## Troubleshooting

### Common Issues

**Docker/Kubernetes Issues**
```bash
# Reset local Kubernetes cluster
docker desktop --reset-kubernetes

# Clean up old Docker images
docker system prune -a
```

**Database Connection Issues**
```bash
# Check if database is running
docker ps | grep postgres

# Reset database
docker-compose -f docker/dev-infrastructure.yml down -v
docker-compose -f docker/dev-infrastructure.yml up -d
./scripts/init-databases.sh
```

**GIS Tileset Issues**
```bash
# Verify tilesets were imported correctly
docker exec -it postgis psql -U postgres -d gis_dev -c "SELECT count(*) FROM spatial_ref_sys;"

# Reimport if necessary
./scripts/import-tilesets.sh ~/dev/gis-data --force
```

## Next Steps

Once everything is running:

1. Check out the [Dev Guide](./dev-guide.md) for daily workflow
2. Explore the [Architecture Overview](./architecture-overview.md)
3. Review [API Documentation](http://localhost:8080/api-docs)

---

**Note:** This setup is for development only. Production deployment uses different secrets, certificates, and configuration, managed through our GitOps pipeline.
