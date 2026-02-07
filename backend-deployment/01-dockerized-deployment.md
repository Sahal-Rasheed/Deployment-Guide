# Dockerized Backend Deployment with CI/CD

> **Goal**: Deploy a production-ready Dockerized backend (Django DRF / FastAPI) with automated CI/CD pipeline using GitHub Actions and container registry.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Project Structure](#2-project-structure)
- [3. Dockerfile for Production](#3-dockerfile-for-production)
- [4. Docker Compose for Production](#4-docker-compose-for-production)
- [5. Environment Variables & Secrets](#5-environment-variables--secrets)
- [6. GitHub Container Registry (GHCR)](#6-github-container-registry-ghcr)
- [7. GitHub Actions CI/CD Pipeline](#7-github-actions-cicd-pipeline)
- [8. Server Setup for Deployment](#8-server-setup-for-deployment)
- [9. Deployment Scripts](#9-deployment-scripts)
- [10. Complete Workflow Example](#10-complete-workflow-example)
- [11. Health Checks & Monitoring](#11-health-checks--monitoring)
- [12. Rollback Strategy](#12-rollback-strategy)
- [13. Troubleshooting](#13-troubleshooting)
- [14. Quick Reference](#14-quick-reference)

---

## 1. Overview

### The Complete CI/CD Flow:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DEVELOPMENT                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Developer pushes code to GitHub (main branch)                              │
│                              │                                               │
│                              ▼                                               │
│   ┌──────────────────────────────────────────────────────────┐              │
│   │              GitHub Actions Triggered                     │              │
│   │  1. Checkout code                                         │              │
│   │  2. Run tests                                             │              │
│   │  3. Build Docker image                                    │              │
│   │  4. Push to GitHub Container Registry (GHCR)              │              │
│   │  5. SSH to VPS and deploy                                 │              │
│   └──────────────────────────────────────────────────────────┘              │
│                              │                                               │
└──────────────────────────────┼───────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PRODUCTION (VPS)                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│   │   Pull latest   │───▶│  Docker Compose │───▶│  App running    │         │
│   │   image from    │    │  up -d          │    │  on port 8000   │         │
│   │   GHCR          │    │                 │    │                 │         │
│   └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                                          │                   │
│                                                          ▼                   │
│                                                 ┌─────────────────┐         │
│                                                 │     Nginx       │         │
│                                                 │  (Reverse Proxy)│         │
│                                                 │   Port 80/443   │         │
│                                                 └─────────────────┘         │
│                                                          │                   │
│                                                          ▼                   │
│                                                      INTERNET                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Approach?

| Aspect                 | Benefit                                             |
| ---------------------- | --------------------------------------------------- |
| **Container Registry** | Versioned images, easy rollback, no build on server |
| **GitHub Actions**     | Free CI/CD, integrated with GitHub, easy secrets    |
| **Docker Compose**     | Consistent deployments, easy service management     |
| **SSH Deployment**     | Simple, secure, no additional tools needed          |

### Prerequisites:

- ✅ VPS with Docker installed ([03-docker.md](../server-setup/03-docker.md))
- ✅ Nginx configured ([05-nginx.md](../server-setup/05-ngnix.md))
- ✅ SSL certificates ([06-domain-ssl.md](../server-setup/06-domain-ssl.md))
- ✅ GitHub Deploy Key ([04-git.md](../server-setup/04-git.md))
- ✅ Domain pointing to your server

---

## 2. Project Structure

### Recommended Project Layout:

```
myapp/
├── .github/
│   └── workflows/
│       └── deploy.yml          # CI/CD pipeline
├── src/                        # Your application code
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py             # FastAPI entry point
│   │   └── ...
│   ├── manage.py               # Django entry point
│   └── requirements.txt
├── Dockerfile                  # Production Dockerfile
├── docker-compose.yml          # Local development
├── docker-compose.prod.yml     # Production compose
├── .env.example                # Example environment variables
├── .dockerignore               # Files to exclude from Docker build
└── README.md
```

### .dockerignore:

```bash
# .dockerignore
.git
.gitignore
.github
.env
.env.*
*.md
*.log
__pycache__
*.pyc
*.pyo
.pytest_cache
.coverage
htmlcov
.venv
venv
node_modules
.DS_Store
*.swp
*.swo
docker-compose*.yml
Dockerfile*
```

---

## 3. Dockerfile for Production

The Dockerfile examples below use **uv** as the package manager for faster builds and smaller images. It follows a **multi-stage** build approach to separate the build environment from the production environment. Adjust the application paths and commands as needed for your specific project structure.

### Django DRF Dockerfile (with uv):

> Note: For FastAPI, the Dockerfile is similar you can make adjustments to the entrypoint and command as needed.

```dockerfile
# Dockerfile
# Reference: https://docs.astral.sh/uv/guides/integration/docker/

# ==================== BUILD STAGE ====================
# This stage is responsible ONLY for dependency resolution and building the virtual environment.

FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS builder

# Enable bytecode compilation and copy mode
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy

# Omit development dependencies
ENV UV_NO_DEV=1

# Force UV to use Sytem Python
ENV UV_PYTHON_DOWNLOADS=0

# Set working directory
WORKDIR /app

# Install only runtime system libraries required by your Python dependencies. Compilation tools should exist only in the builder stage. (like build-essential, libpq-dev for psycopg2, etc.) if any add it in build stage here.
RUN apt-get update && apt-get install -y --no-install-recommends build-essential libpq-dev && rm -rf /var/lib/apt/lists/*

# Install dependencies based on uv lockfile
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project

# ==================== PRODUCTION STAGE ====================
# This stage contains the final runtime image without uv or build tools. (Only Python, the virtual environment, and the application code)

# IMPORTANT: Python version and distro must match the builder
FROM python:3.12-slim-bookworm AS production

# Create non-root user to run the application for security (Container User)
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH="/app/.venv/bin:$PATH"

WORKDIR /app

# Install ONLY runtime libraries, (like libpq for PostgreSQL, curl for health checks, etc.) - Like packages needed to run the app, not build it
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy the pre-built virtual environment from the builder stage
COPY --from=builder /app/.venv /app/.venv

# Copy application code
# --chown ensures files are owned by the non-root user (user we created for security)
COPY --chown=appuser:appgroup . .

# Copy and set entrypoint
COPY --chown=appuser:appgroup entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

# Switch to non-root user
USER appuser

EXPOSE 8000

# Container Health check (Optional, If using make sure u have this /health endpoint in your app. Also if you prefer to have this in docker-compose (healthcheck:) instead then you can remove this from Dockerfile
# HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
#     CMD curl -f http://localhost:8000/health/ || exit 1

# Handles migrations & startup logic
ENTRYPOINT ["/app/entrypoint.sh"]

# Default app run command (can be overridden from docker-compose) if multiple environments better to keep this in docker-compose instead of Dockerfile
CMD ["gunicorn", "core.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

### entrypoint.sh:

> **Note**: This script (we attached to our Dockerfile) runs every time the container starts. It handles database migrations and collects static files (for Django) before starting the server. Adjust commands as needed for FastAPI or other frameworks.

```bash
#!/bin/sh
set -e

echo "🚀 Starting application..."

# Run migrations (Django only)
if [ -f "manage.py" ]; then
    echo "📦 Running migrations..."
    python manage.py migrate --noinput

    # Collect static files (only in production)
    if [ "$DEBUG" != "True" ]; then
        echo "📁 Collecting static files..."
        python manage.py collectstatic --noinput
    fi
fi

echo "✅ Starting server..."
exec "$@"
```

> **Note**: Make sure `entrypoint.sh` has Unix line endings (LF, not CRLF). You can convert with: `sed -i 's/\r$//' entrypoint.sh`

### Why Multi-Stage Build with uv?

| Stage          | Purpose              | Contents                         |
| -------------- | -------------------- | -------------------------------- |
| **Builder**    | Install dependencies | uv, build cache, .venv           |
| **Production** | Run application      | Python, .venv (copied), app code |

Benefits:

- **Smaller image**: No uv, no build tools in final image
- **Faster CI builds**: Layer caching for dependencies
- **Security**: Fewer packages = smaller attack surface

### Dockerfile Best Practices:

| Practice                | Reason                                      |
| ----------------------- | ------------------------------------------- |
| **Multi-stage build**   | Smaller final image (~200MB vs ~800MB)      |
| **uv package manager**  | 10-100x faster than pip                     |
| **Non-root user**       | Security - limits container privileges      |
| **HEALTHCHECK**         | Docker/orchestrator knows if app is healthy |
| **Separate entrypoint** | Clean separation, easy to modify            |
| **Layer caching**       | Mount cache for faster rebuilds             |

---

## 4. Docker Compose for Production

### docker-compose.prod.yml (CI/CD - Pull from Registry):

> Note: This compose file is used on the server and pulls pre-built images from GHCR. This compose.prod is optimized for production for drf, for fastapi or other frameworks you can adjust the service definitions, health checks, db, redis etc as needed. Most of the structure remains the same regardless of framework.

```yaml
# docker-compose.prod.yml
# Used on SERVER - pulls pre-built image from GHCR

services:
  web:
    image: ghcr.io/${GITHUB_USERNAME}/${GITHUB_REPO}:${IMAGE_TAG:-latest}
    container_name: ${PROJECT_NAME:-myapp}-web
    # command: gunicorn core.wsgi --bind 0.0.0.0:8000 --workers=4 ## No need to specify command if it's in Dockerfile CMD, Priority: docker-compose command > Dockerfile CMD
    restart: unless-stopped
    env_file:
      - .env.prod
    environment: # Same as .env.prod but with prefix for docker-compose (optional)
      - DJANGO_SETTINGS_MODULE=core.settings
    ports:
      - "127.0.0.1:8000:8000" # Only localhost (Nginx proxies)
    volumes:
      - static_volume:/app/staticfiles
      - media_volume:/app/mediafiles
      - ./logs:/app/logs
    networks:
      - app-network
    depends_on:
      db:
        condition: service_healthy
    healthcheck: # Optional, make sure you have a /health endpoint in your app
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s # optional
      timeout: 10s
      retries: 3
      start_period: 40s # optional, gives app time to start before health checks begin

  db:
    image: postgres:16-alpine
    container_name: ${PROJECT_NAME:-myapp}-db
    restart: unless-stopped
    env_file:
      - .env.prod
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: ${PROJECT_NAME:-myapp}-redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:

networks:
  app-network:
    driver: bridge
```

### docker-compose.yml (Local Development - Build Locally):

> Note: This compose file is for local development. It builds the image locally and mounts the source code for hot reload. It also exposes database ports for local tools. This is NOT used in production.

```yaml
# docker-compose.yml
# Used for LOCAL DEVELOPMENT - builds image locally

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    image: ${PROJECT_NAME:-myapp}:dev
    container_name: ${PROJECT_NAME:-myapp}-web-dev
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app # Mount source code for hot reload
      - /app/.venv # Don't override .venv since using uv
    env_file:
      - .env
    ports:
      - "${PORT:-8000}:8000"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  db:
    image: postgres:16-alpine
    container_name: ${PROJECT_NAME:-myapp}-db-dev
    env_file:
      - .env
    volumes:
      - postgres_data_dev:/var/lib/postgresql/data
    ports:
      - "5432:5432" # Expose for local tools
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: ${PROJECT_NAME:-myapp}-redis-dev
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  postgres_data_dev:

networks:
  app-network:
    driver: bridge
```

### Key Differences:

| Aspect       | docker-compose.yml (Dev)        | docker-compose.prod.yml (Prod)            |
| ------------ | ------------------------------- | ----------------------------------------- |
| **Image**    | `build: .` (build locally)      | `image: ghcr.io/...` (pull from registry) |
| **Command**  | `runserver` (Django dev server) | `gunicorn`                                |
| **Volumes**  | Mount source code (hot reload)  | Only mount data volumes                   |
| **Ports**    | Exposed to host                 | Only `127.0.0.1` (Nginx proxies)          |
| **DB Ports** | Exposed (for local tools)       | Not exposed (internal only)               |

### .env.prod Example:

```bash
# .env.prod - Production environment variables

# Django
DEBUG=False
SECRET_KEY=your-super-secret-key-generate-this
ALLOWED_HOSTS=api.example.com

# Database
POSTGRES_DB=myapp
POSTGRES_USER=myapp_user
POSTGRES_PASSWORD=super-secure-password-here
DATABASE_URL=postgresql://myapp_user:super-secure-password-here@db:5432/myapp

# Redis
REDIS_URL=redis://redis:6379/0

# CORS
CORS_ALLOWED_ORIGINS=https://app.example.com,https://example.com

# Project (for container naming)
PROJECT_NAME=myapp

# GHCR (for docker-compose to know which image)
GITHUB_USERNAME=yourusername
GITHUB_REPO=myapp
IMAGE_TAG=latest
```

---

## 6. GitHub Container Registry (GHCR)

### Why GHCR?

| Registry                      | Pros                                                     | Cons                                       |
| ----------------------------- | -------------------------------------------------------- | ------------------------------------------ |
| **GitHub Container Registry** | Free for public repos, integrated with GitHub, easy auth | Newer, less features than Docker Hub       |
| Docker Hub                    | Popular, lots of images                                  | Rate limits on free tier, separate account |
| AWS ECR                       | Integrated with AWS                                      | Complex, costs money                       |

We'll use **GHCR** for seamless GitHub integration.

### Understanding GITHUB_TOKEN vs Personal Access Token (PAT)

> **Important**: GitHub provides an automatic `GITHUB_TOKEN` for GitHub Actions workflows. You don't need a PAT for building and pushing images within GitHub Actions!

| Token            | Where Used         | Purpose                     | Setup Required?   |
| ---------------- | ------------------ | --------------------------- | ----------------- |
| **GITHUB_TOKEN** | GitHub Actions     | Build & push images to GHCR | ❌ No - automatic |
| **PAT**          | Server / Local CLI | Pull images from GHCR       | ✅ Yes - manual   |

**Reference**: [GitHub Docs - Publishing packages with GitHub Actions](https://docs.github.com/en/packages/managing-github-packages-using-github-actions-workflows/publishing-and-installing-a-package-with-github-actions)

### Step 6.1: Create Personal Access Token (PAT) - For Server Access

> **Note**: This PAT is needed for your **server** to pull images from GHCR. The GitHub Actions workflow uses the automatic `GITHUB_TOKEN` for pushing images.

1. Go to **GitHub** → **Settings** → **Developer settings**
2. Click **Personal access tokens** → **Tokens (classic)**
3. Click **Generate new token (classic)**
4. Configure:
   - **Note**: `ghcr-server-pull-token`
   - **Expiration**: 90 days (or longer, or no expiration for server)
   - **Scopes**: Select:
     - `read:packages` (pull images) - **Required**
     - `write:packages` (only if you need to push from CLI)
5. Click **Generate token**
6. **Copy the token immediately!**

### Step 6.2: Add Secrets to Repository

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add these secrets:

| Name             | Value                                           | Used For                      |
| ---------------- | ----------------------------------------------- | ----------------------------- |
| `GHCR_PAT`       | Your Personal Access Token                      | Server pulls images from GHCR |
| `SERVER_HOST`    | Your server IP or domain (e.g., `123.45.67.89`) | SSH connection                |
| `SERVER_USER`    | Your deploy user (e.g., `deploy`)               | SSH connection                |
| `SERVER_SSH_KEY` | Your server's private SSH key (see below)       | SSH connection                |

> **Note**: We name it `GHCR_PAT` to distinguish from `GITHUB_TOKEN`. The `GITHUB_TOKEN` is automatically available and used in the build job - no setup needed!

### Step 6.3: Generate Server SSH Key for GitHub Actions

On your **local machine** (not server):

```bash
# Generate a dedicated key for GitHub Actions deployment
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy -N ""

# Display the private key (add this to GitHub Secrets as SERVER_SSH_KEY)
cat ~/.ssh/github_actions_deploy

# Display the public key (add this to server's authorized_keys)
cat ~/.ssh/github_actions_deploy.pub
```

On your **server**:

```bash
# Add the public key to authorized_keys (.)
echo "ssh-ed25519 AAAAC3... github-actions-deploy" >> ~/.ssh/authorized_keys
```

### Step 6.4: Test GHCR Login (Optional)

**From Local Machine (for pushing):**

```bash
# Login to GHCR with PAT (needs write:packages scope)
export GHCR_PAT="ghp_your_token_here"
echo $GHCR_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# Build and push test
docker build -t ghcr.io/yourusername/myapp:test .
docker push ghcr.io/yourusername/myapp:test
```

**From Server (for pulling):**

```bash
# Login to GHCR with PAT (needs read:packages scope)
export GHCR_PAT="ghp_your_token_here"
echo $GHCR_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# Pull image
docker pull ghcr.io/yourusername/myapp:latest
```

### Token Usage Summary:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TOKEN USAGE                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  GitHub Actions (Build & Push):                                              │
│  └─► Uses: GITHUB_TOKEN (automatic, no setup)                               │
│                                                                              │
│  Server (Pull Images):                                                       │
│  └─► Uses: GHCR_PAT (Personal Access Token, stored in secrets)              │
│                                                                              │
│  Local CLI (Optional):                                                       │
│  └─► Uses: PAT with write:packages (for pushing) or read:packages (pulling) │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. GitHub Actions CI/CD Pipeline

### .github/workflows/deploy.yml:

```yaml
# .github/workflows/deploy.yml

name: Build and Deploy

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ==================== TEST ====================
  # If you have tests, run them here before building the image.
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: uv sync --locked

      - name: Run tests
        env:
          DATABASE_URL: postgresql://test_user:test_pass@localhost:5432/test_db
          SECRET_KEY: test-secret-key
          DEBUG: "True"
        run: uv run pytest --cov=. --cov-report=xml

      # Optional: Upload coverage to Codecov
      # - name: Upload coverage
      #   uses: codecov/codecov-action@v4

  # ==================== BUILD & PUSH ====================
  build:
    name: Build and Push Image
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
      image_digest: ${{ steps.build.outputs.digest }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }} # or your GitHub username (this should match the user associated with the GITHUB_TOKEN). Keep username in secrets if so.
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64

      - name: Image digest
        run: echo "Image digest - ${{ steps.build.outputs.digest }}"

  # ==================== DEPLOY ====================
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    environment:
      name: production
      url: https://api.example.com

    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd ~/apps/myapp

            # Login to GHCR (uses PAT because server is external to GitHub Actions)
            echo ${{ secrets.GHCR_PAT }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin

            # Pull latest image
            docker compose -f docker-compose.prod.yml pull

            # Deploy with zero-downtime
            docker compose -f docker-compose.prod.yml up -d --remove-orphans

            # Run migrations (Django) or any other post-deploy commands, Currently we are handling migrations in entrypoint.sh so no need to run here.

            # Clean up old images
            docker image prune -af --filter "until=24h"

            # Verify deployment
            sleep 10
            curl -f http://localhost:8000/health/ || exit 1

      - name: Notify on success
        if: success()
        run: echo "✅ Deployment successful!"

      - name: Notify on failure
        if: failure()
        run: echo "❌ Deployment failed!"
```

### Workflow Explained:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              GITHUB ACTIONS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐                                                            │
│  │   TEST JOB  │  ← Runs on every push/PR                                   │
│  │             │                                                            │
│  │  • Checkout │                                                            │
│  │  • Setup Python                                                          │
│  │  • Install deps                                                          │
│  │  • Run pytest                                                            │
│  └──────┬──────┘                                                            │
│         │ (only on main branch push)                                        │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  BUILD JOB  │                                                            │
│  │             │                                                            │
│  │  • Checkout │                                                            │
│  │  • Login GHCR                                                            │
│  │  • Build image                                                           │
│  │  • Push to GHCR                                                          │
│  └──────┬──────┘                                                            │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │ DEPLOY JOB  │                                                            │
│  │             │                                                            │
│  │  • SSH to server                                                         │
│  │  • Pull image                                                            │
│  │  • docker compose up                                                     │
│  │  • Run migrations                                                        │
│  │  • Health check                                                          │
│  └─────────────┘                                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### For FastAPI (simpler workflow):

```yaml
# .github/workflows/deploy.yml (FastAPI version)

name: Build and Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: uv sync --locked
      - run: uv run pytest

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd ~/apps/myapp
            # Uses PAT because server is external to GitHub Actions
            echo ${{ secrets.GHCR_PAT }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker compose -f docker-compose.prod.yml pull
            docker compose -f docker-compose.prod.yml up -d
            docker image prune -af --filter "until=24h"
```

---

## 8. Server Setup for Deployment

### Step 8.1: Create Application Directory

```bash
# SSH to your server
ssh deploy@your-server-ip

# Create apps directory in user's home folder
mkdir -p ~/apps/myapp
cd ~/apps/myapp
```

### Directory Structure on Server:

```
/home/deploy/
├── .ssh/
│   └── authorized_keys          # Contains GitHub Actions deploy key
└── apps/
    └── myapp/
        ├── docker-compose.prod.yml
        ├── .env.prod
        └── (volumes mounted here by Docker)
```

> **Why ~/apps?**:
>
> - User-owned, no sudo needed
> - Easy to manage multiple apps
> - Clear separation from system files
> - Follows principle of least privilege

### Step 8.2: Create Production Files on Server

```bash
cd ~/apps/myapp

# Create docker-compose.prod.yml
nano docker-compose.prod.yml
# (paste the content from section 4)

# Create .env.prod
nano .env.prod
# (paste the content from section 5, with real values)
```

### Step 8.3: Initial Manual Deployment (First Time Only)

```bash
# Login to GHCR (use your Personal Access Token with read:packages scope)
echo "YOUR_GHCR_PAT" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# Pull and start
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d

# Check logs
docker compose -f docker-compose.prod.yml logs -f

# Run migrations (Django)
docker compose -f docker-compose.prod.yml exec web python manage.py migrate
```

### Step 8.4: Configure Nginx

Create Nginx config for your API (see [05-nginx.md](../server-setup/05-ngnix.md) for details):

```bash
sudo nano /etc/nginx/sites-available/api.example.com
```

```nginx
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # SSL (managed by Certbot or use wildcard)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Logging
    access_log /var/log/nginx/api.example.com.access.log;
    error_log /var/log/nginx/api.example.com.error.log;

    # Proxy to Docker container
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Static files (Django)
    location /static/ {
        alias /home/deploy/apps/myapp/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Media files (Django)
    location /media/ {
        alias /home/deploy/apps/myapp/mediafiles/;
        expires 7d;
    }
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 9. Deployment Scripts

### Optional: Local Deploy Script

Create `scripts/deploy.sh` for manual deployments:

```bash
#!/bin/bash
# scripts/deploy.sh - Manual deployment script

set -e

SERVER_USER="deploy"
SERVER_HOST="your-server-ip"
APP_DIR="~/apps/myapp"

echo "🚀 Starting deployment..."

# SSH and deploy
ssh ${SERVER_USER}@${SERVER_HOST} << 'EOF'
    cd ~/apps/myapp

    echo "📥 Pulling latest image..."
    docker compose -f docker-compose.prod.yml pull

    echo "🔄 Updating containers..."
    docker compose -f docker-compose.prod.yml up -d --remove-orphans

    echo "🗃️ Running migrations..."
    docker compose -f docker-compose.prod.yml exec -T web python manage.py migrate --noinput

    echo "🧹 Cleaning up..."
    docker image prune -af --filter "until=24h"

    echo "✅ Deployment complete!"
EOF
```

### Server-side Management Script

Create on server at `~/apps/myapp/manage.sh`:

```bash
#!/bin/bash
# ~/apps/myapp/manage.sh - Server management script

set -e
cd ~/apps/myapp

case "$1" in
    up)
        docker compose -f docker-compose.prod.yml up -d
        ;;
    down)
        docker compose -f docker-compose.prod.yml down
        ;;
    restart)
        docker compose -f docker-compose.prod.yml restart
        ;;
    logs)
        docker compose -f docker-compose.prod.yml logs -f ${2:-}
        ;;
    shell)
        docker compose -f docker-compose.prod.yml exec web /bin/bash
        ;;
    migrate)
        docker compose -f docker-compose.prod.yml exec web python manage.py migrate
        ;;
    status)
        docker compose -f docker-compose.prod.yml ps
        ;;
    pull)
        docker compose -f docker-compose.prod.yml pull
        ;;
    update)
        docker compose -f docker-compose.prod.yml pull
        docker compose -f docker-compose.prod.yml up -d --remove-orphans
        docker image prune -af --filter "until=24h"
        ;;
    *)
        echo "Usage: $0 {up|down|restart|logs|shell|migrate|status|pull|update}"
        exit 1
        ;;
esac
```

Make executable:

```bash
chmod +x ~/apps/myapp/manage.sh

# Usage examples:
./manage.sh up
./manage.sh logs web
./manage.sh migrate
./manage.sh update
```

---

## 10. Complete Workflow Example

### End-to-End: From Code to Production

```
1. DEVELOPER (Local)
   │
   │  git push origin main
   │
   ▼
2. GITHUB
   │
   │  Push triggers GitHub Actions workflow
   │
   ▼
3. GITHUB ACTIONS (Test Job)
   │
   │  ✓ Checkout code
   │  ✓ Setup Python 3.12
   │  ✓ Install dependencies
   │  ✓ Run pytest (with test database)
   │
   ▼
4. GITHUB ACTIONS (Build Job)
   │
   │  ✓ Checkout code
   │  ✓ Setup Docker Buildx
   │  ✓ Login to GHCR
   │  ✓ Build Docker image (multi-stage)
   │  ✓ Push image to ghcr.io/user/repo:latest
   │  ✓ Push image to ghcr.io/user/repo:abc123 (commit SHA)
   │
   ▼
5. GITHUB ACTIONS (Deploy Job)
   │
   │  ✓ SSH to production server
   │  ✓ cd ~/apps/myapp
   │  ✓ docker login to GHCR
   │  ✓ docker compose pull (download new image)
   │  ✓ docker compose up -d (restart with new image)
   │  ✓ Run database migrations
   │  ✓ Cleanup old images
   │  ✓ Health check
   │
   ▼
6. PRODUCTION SERVER
   │
   │  ✓ New container running
   │  ✓ Nginx proxying traffic
   │  ✓ SSL termination
   │
   ▼
7. USERS
   │
   │  Access https://api.example.com
```

### First-Time Setup Checklist:

```bash
# ========== GITHUB SETUP ==========
# 1. Create repository secrets (Settings → Secrets → Actions)
#    - GHCR_PAT: Personal Access Token (for server to pull images)
#    - SERVER_HOST: Your server IP
#    - SERVER_USER: deploy
#    - SERVER_SSH_KEY: Private key content
#    Note: GITHUB_TOKEN is automatic - no setup needed for build/push!

# 2. Create .github/workflows/deploy.yml in your repo

# ========== SERVER SETUP ==========
# 3. SSH to server
ssh deploy@your-server-ip

# 4. Create app directory
mkdir -p ~/apps/myapp
cd ~/apps/myapp

# 5. Create docker-compose.prod.yml
nano docker-compose.prod.yml

# 6. Create .env.prod with real values
nano .env.prod

# 7. Add GitHub Actions SSH key to authorized_keys
echo "ssh-ed25519 AAAA... github-actions-deploy" >> ~/.ssh/authorized_keys

# 8. Initial pull (first time only)
echo "ghp_xxxx" | docker login ghcr.io -u yourusername --password-stdin
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d

# 9. Configure Nginx
sudo nano /etc/nginx/sites-available/api.example.com
sudo ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# 10. Setup SSL (if not using wildcard)
sudo certbot --nginx -d api.example.com

# ========== TEST DEPLOYMENT ==========
# 11. Push to main branch and watch GitHub Actions
git push origin main

# 12. Check deployment
curl https://api.example.com/health/
```

---

## 11. Health Checks & Monitoring

### Add Health Check Endpoint

**Django:**

```python
# views.py
from django.http import JsonResponse
from django.db import connection

def health_check(request):
    try:
        # Check database connection
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")

        return JsonResponse({
            "status": "healthy",
            "database": "connected"
        })
    except Exception as e:
        return JsonResponse({
            "status": "unhealthy",
            "error": str(e)
        }, status=500)

# urls.py
urlpatterns = [
    path('health/', views.health_check, name='health_check'),
    # ...
]
```

**FastAPI:**

```python
# main.py
from fastapi import FastAPI
from sqlalchemy import text

app = FastAPI()

@app.get("/health")
async def health_check():
    try:
        # Check database (example with SQLAlchemy)
        async with async_session() as session:
            await session.execute(text("SELECT 1"))

        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

### Monitoring Commands:

```bash
# Check container status
docker compose -f docker-compose.prod.yml ps

# Check container health
docker inspect --format='{{.State.Health.Status}}' myapp-web

# View logs
docker compose -f docker-compose.prod.yml logs -f web

# Check resource usage
docker stats

# Test health endpoint
curl http://localhost:8000/health/
```

---

## 12. Rollback Strategy

### Rollback to Previous Version

Images are tagged with commit SHA, making rollback easy:

```bash
# List available image tags
docker images ghcr.io/yourusername/myapp

# Or check GHCR on GitHub (Packages tab)
```

### Rollback Steps:

```bash
cd ~/apps/myapp

# 1. Update IMAGE_TAG in .env.prod (or docker-compose)
nano .env.prod
# Change: IMAGE_TAG=abc123  (previous commit SHA)

# 2. Pull the specific version
docker compose -f docker-compose.prod.yml pull

# 3. Deploy the old version
docker compose -f docker-compose.prod.yml up -d

# 4. Verify
curl http://localhost:8000/health/
```

### Quick Rollback Script:

```bash
#!/bin/bash
# rollback.sh

if [ -z "$1" ]; then
    echo "Usage: ./rollback.sh <commit-sha>"
    exit 1
fi

cd ~/apps/myapp
export IMAGE_TAG=$1

docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d

echo "Rolled back to $1"
```

---

## 13. Troubleshooting

### Build Failures:

```bash
# Check GitHub Actions logs
# Go to repo → Actions → Click failed workflow → Click failed job

# Common issues:
# - Missing requirements.txt
# - Dockerfile syntax errors
# - Test failures
```

### Deployment Failures:

```bash
# SSH to server and check manually
ssh deploy@your-server-ip

cd ~/apps/myapp

# Check container status
docker compose -f docker-compose.prod.yml ps

# Check logs
docker compose -f docker-compose.prod.yml logs web

# Check if image was pulled
docker images | grep myapp

# Try manual pull
docker compose -f docker-compose.prod.yml pull
```

### Container Won't Start:

```bash
# Check logs for errors
docker compose -f docker-compose.prod.yml logs web

# Common issues:
# - Database connection errors → Check DB container is running
# - Missing environment variables → Check .env.prod
# - Port already in use → Check with: sudo ss -tlnp | grep 8000
```

### GHCR Authentication Issues:

```bash
# Test login manually (use your PAT)
echo $GHCR_PAT | docker login ghcr.io -u yourusername --password-stdin

# If token expired, generate new one on GitHub
# Update GHCR_PAT secret in repository settings
```

### SSH Connection Issues:

```bash
# Test SSH from local machine
ssh -i ~/.ssh/github_actions_deploy deploy@your-server-ip

# Check authorized_keys on server
cat ~/.ssh/authorized_keys

# Check SSH key permissions
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

---

## 14. Quick Reference

### GitHub Secrets Required:

| Secret           | Description                                 |
| ---------------- | ------------------------------------------- |
| `GHCR_PAT`       | PAT with read:packages (for server to pull) |
| `SERVER_HOST`    | Server IP or domain                         |
| `SERVER_USER`    | SSH user (e.g., deploy)                     |
| `SERVER_SSH_KEY` | Private key for SSH                         |

> **Note**: `GITHUB_TOKEN` is automatically available in GitHub Actions - no setup needed!

### Server File Locations:

```
~/apps/myapp/
├── docker-compose.prod.yml    # Production compose file
├── .env.prod                  # Environment variables
└── manage.sh                  # Management script (optional)
```

### Common Commands:

```bash
# ===== LOCAL =====
# Push to trigger deployment
git push origin main

# ===== SERVER =====
# Check status
docker compose -f docker-compose.prod.yml ps

# View logs
docker compose -f docker-compose.prod.yml logs -f web

# Restart
docker compose -f docker-compose.prod.yml restart

# Manual update
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d

# Run migrations
docker compose -f docker-compose.prod.yml exec web python manage.py migrate

# Shell into container
docker compose -f docker-compose.prod.yml exec web /bin/bash

# Cleanup
docker system prune -af
```

### Deployment Flow:

```
git push → GitHub Actions → Build Image → Push to GHCR → SSH Deploy → Done!
```

---

## ✅ Production Checklist

- [ ] Repository secrets configured (GHCR*TOKEN, SERVER*\*, etc.)
- [ ] Dockerfile with multi-stage build
- [ ] docker-compose.prod.yml on server
- [ ] .env.prod with secure secrets on server
- [ ] GitHub Actions SSH key in authorized_keys
- [ ] GitHub Actions workflow in .github/workflows/
- [ ] Nginx configured for domain
- [ ] SSL certificate configured
- [ ] Health check endpoint working
- [ ] First deployment tested manually
- [ ] CI/CD pipeline tested with push to main

---

**Next Guide**: [Non-Dockerized Deployment with Systemd →](./02-systemd-deployment.md)

**Related Guides**:

- [Nginx Configuration](../server-setup/05-ngnix.md)
- [SSL Setup](../server-setup/06-domain-ssl.md)
- [Git & Deploy Keys](../server-setup/04-git.md)
