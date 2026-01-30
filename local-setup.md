# Local Development Setup

Get from zero to running in under 30 minutes.

---

## Prerequisites

| Tool | Version | Installation |
|------|---------|--------------|
| Git | 2.35+ | [git-scm.com](https://git-scm.com) |
| Docker Desktop | 4.15+ | With Kubernetes enabled |
| Node.js | 18+ | Via nvm |
| Python | 3.10+ | Via pyenv |
| VS Code | Latest | [code.visualstudio.com](https://code.visualstudio.com) |
| kubectl | Latest | `brew install kubectl` / `choco install kubernetes-cli` |

---

## Platform-Specific Setup

### macOS

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install core tools
brew install git node@18 python@3.10 kubectl
brew install --cask docker visual-studio-code postman

# Enable Kubernetes in Docker Desktop: Settings → Kubernetes → Enable Kubernetes
```

### Windows

```powershell
# Install Chocolatey
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# Install tools
choco install -y git nodejs-lts python kubernetes-cli docker-desktop visualstudiocode postman

# Enable Hyper-V (required for Docker)
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

### Linux (Ubuntu/Debian)

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install nvm and Node.js
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.nvm/nvm.sh && nvm install 18
```

---

## Repository Setup

```bash
# Create dev directory
mkdir -p ~/dev/platform
cd ~/dev/platform

# Clone repositories
git clone git@github.com:company/core-platform.git
git clone git@github.com:company/edtech-services.git
git clone git@github.com:company/gis-services.git
git clone git@github.com:company/frontend.git

# Initialize submodules
find . -type d -depth 1 -exec bash -c "cd '{}' && git submodule update --init --recursive" \;
```

---

## Environment Configuration

```bash
# Copy example env files
find . -name ".env.example" -exec bash -c 'cp "$1" "${1%.example}"' _ {} \;

# Generate dev certificates
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout dev-cert-key.pem -out dev-cert.pem \
  -subj "/CN=localhost"
```

### Required Environment Variables

**Core Platform (.env)**
```bash
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/core_platform_dev
REDIS_URL=redis://localhost:6379/0
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=dev-assets
JWT_SECRET=development_secret_change_in_production
```

**EdTech Services (.env)**
```bash
CORE_PLATFORM_URL=http://localhost:3000
OPENAI_API_KEY=your_api_key_here
FFMPEG_THREADS=4
```

**GIS Services (.env)**
```bash
POSTGIS_URL=postgresql://postgres:postgres@localhost:5433/gis_dev
TITILER_ENDPOINT=http://localhost:8000
MAX_ZOOM_LEVEL=16
```

---

## Database Setup

```bash
# Start infrastructure (PostgreSQL, PostGIS, Redis, MinIO)
docker-compose -f docker/dev-infrastructure.yml up -d

# Initialize databases
./scripts/init-databases.sh
```

---

## GIS Data Setup

```bash
# Create GIS data directory
mkdir -p ~/dev/gis-data

# Download sample tilesets
curl -L https://github.com/company/sample-tilesets/releases/download/v1.0.0/sample-tilesets.tar.gz | tar -xz -C ~/dev/gis-data

# Import into PostGIS
cd ~/dev/platform/gis-services
./scripts/import-tilesets.sh ~/dev/gis-data
```

---

## Kubernetes Setup

```bash
# Create development namespace
kubectl create namespace platform-dev

# Apply dev configs
kubectl apply -f k8s/dev-configs.yaml -n platform-dev

# Configure local registry
kubectl apply -f k8s/local-registry.yaml
```

---

## Build and Run

```bash
# Build all Docker images
cd ~/dev/platform
./scripts/build-all.sh

# Start development environment
./scripts/start-dev.sh
```

**This script:**
- Builds all Docker images with dev tags
- Deploys to local Kubernetes
- Sets up port forwarding for each service
- Configures hot reloading

---

## Verification

```bash
./scripts/verify-setup.sh
```

**Expected output:**
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

---

## Troubleshooting

### Docker/Kubernetes

```bash
# Reset local Kubernetes
docker desktop --reset-kubernetes

# Clean up old Docker images
docker system prune -a

# Check Kubernetes pods
kubectl get pods -n platform-dev
```

### Database

```bash
# Check if database is running
docker ps | grep postgres

# Reset database
docker-compose -f docker/dev-infrastructure.yml down -v
docker-compose -f docker/dev-infrastructure.yml up -d
./scripts/init-databases.sh
```

### GIS Tilesets

```bash
# Verify tilesets imported
docker exec -it postgis psql -U postgres -d gis_dev -c "SELECT count(*) FROM spatial_ref_sys;"

# Reimport if necessary
./scripts/import-tilesets.sh ~/dev/gis-data --force
```

---

## Next Steps

1. Review [Dev Guide](./dev-guide.md) for daily workflow
2. Explore [Architecture Overview](./architecture-overview.md)
3. Check [API Documentation](http://localhost:8080/api-docs)

---

> **Note:** This setup is for development only. Production uses different secrets, certificates, and GitOps-managed configuration.
