# Docker Installation & Setup

> **Goal**: Install Docker and Docker Compose for containerized deployments. Reference:
https://docs.docker.com/engine/install/ubuntu/

## Table of Contents
- [1. Why Docker?](#1-why-docker)
- [2. Uninstall Old Versions](#2-uninstall-old-versions)
- [3. Install Docker Engine](#3-install-docker-engine)
- [4. Post-Installation Steps](#4-post-installation-steps)
- [5. Install Docker Compose](#5-install-docker-compose)
- [6. Verify Installation](#6-verify-installation)
- [7. Docker Basics](#7-docker-basics)

---

## 1. Why Docker?

### Benefits for Deployment:
- **Consistency**: Same environment in dev, staging, production
- **Isolation**: Apps run in isolated containers
- **Portability**: Run anywhere Docker is installed
- **Scalability**: Easy to scale horizontally
- **Version Control**: Image versioning and rollback

---

## 2. Uninstall Old Versions

Remove any old Docker installations:

```bash
# Remove old versions
sudo apt remove docker docker-engine docker.io containerd runc

# It's OK if apt reports that none of these packages are installed
```

---

## 3. Install Docker Engine
https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

### Step 3.1: Set up Docker's apt repository

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
bash - c 'sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF'
```

### Step 3.2: Install Docker Engine

```bash
# Update apt with new repository
sudo apt update

# Install Docker Engine, CLI, and plugins
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Step 3.3: Verify Docker is running

```bash
# Docker service starts automatically after installation. 
# To verify that Docker is running, use:
sudo systemctl status docker

# Should show: Active: active (running)

# Some systems may have this behavior disabled and will require a manual start:
sudo systemctl start docker
```

---

## 4. Post-Installation Steps
https://docs.docker.com/engine/install/linux-postinstall/

### Step 4.1: Run Docker without sudo

By default, Docker requires sudo. Let's fix that:

```bash
# Create docker group (might already exist)
sudo groupadd docker

# Add your user to the docker group
sudo usermod -aG docker $USER
# OR
sudo adduser $USER docker

# Apply new group membership
newgrp docker

# OR log out and log back in
```

### Step 4.2: Verify non-root access

```bash
# Run without sudo
docker run hello-world
```

Expected output:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

### Step 4.3: Configure Docker to start on boot

```bash
# On Debian and Ubuntu, the Docker service starts on boot by default. To automatically start Docker and containerd on boot for other Linux distributions using systemd, run the following commands:
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# To stop this behavior, use `disable` instead.
```

---

## 5. Install Docker Compose

Docker Compose V2 is installed as a Docker plugin (already done in step 3.2).

### Verify Docker Compose:

```bash
# Check version (note: it's 'docker compose' not 'docker-compose')
docker compose version
# Output: Docker Compose version v2.x.x
```

### Using Docker Compose:

```bash
# New syntax (recommended)
docker compose up -d
docker compose down
docker compose logs

# Old syntax
docker-compose up -d
```

---

## 6. Verify Installation

### Complete verification:

```bash
# Docker version
docker version

# Docker system info
docker info

# Docker Compose version
docker compose version

# Test run
docker run hello-world

# List containers
docker ps -a

# Clean up test container (optional)
docker rm $(docker ps -aq --filter "ancestor=hello-world")
docker rmi hello-world
```

---

## 7. Docker Basics

> We can use tools like `lazydocker` for easier management, but below are essential commands. In applications, we'll use these much easier via Docker Compose.

### Essential Commands:

```bash
# ========== IMAGES ==========
docker images                    # List images
docker pull nginx                # Pull image from Docker Hub
docker rmi nginx                 # Remove image
docker build -t myapp .          # Build image from Dockerfile

# ========== CONTAINERS ==========
docker ps                        # List running containers
docker ps -a                     # List all containers
docker run -d nginx              # Run container in background
docker run -d -p 80:80 nginx     # Run with port mapping
docker run -d --name web nginx   # Run with custom name
docker stop web                  # Stop container
docker start web                 # Start stopped container
docker restart web               # Restart container
docker rm web                    # Remove container
docker rm -f web                 # Force remove running container

# ========== LOGS & EXEC ==========
docker logs web                  # View container logs
docker logs -f web               # Follow logs (live)
docker exec -it web bash         # Enter container shell

# ========== CLEANUP ==========
docker system prune              # Remove unused data
docker system prune -a           # Remove all unused data
docker volume prune              # Remove unused volumes
```

### Docker Run Options Explained:

```bash
docker run \
  -d \                          # Detached mode (background)
  --name myapp \                # Container name
  -p 8000:8000 \                # Port mapping (host:container)
  -v /host/path:/container/path \ # Volume mount
  -e DATABASE_URL=postgres://... \ # Environment variable
  --restart unless-stopped \    # Restart policy
  myimage:latest                # Image to run
```

---

## 📌 Quick Reference

```bash
# Service management
sudo systemctl start docker   # Start Docker
sudo systemctl stop docker    # Stop Docker
sudo systemctl restart docker # Restart Docker
sudo systemctl status docker  # Check status
sudo journalctl -u docker.service     # View logs
sudo systemctl enable docker.service  # Enable on boot

# Common operations
docker system df                # Disk usage
docker ps -a                    # All containers
docker images                   # All images
docker compose up -d            # Start services
docker compose down             # Stop services
docker compose logs -f          # Follow logs
docker system prune -a          # Clean everything
```

---
