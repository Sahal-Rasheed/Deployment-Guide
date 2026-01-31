# Git & GitHub Deploy Keys Setup

> **Goal**: Install Git, configure GitHub Deploy Keys for private repository access on servers, and set up for CI/CD automation.

## Table of Contents

- [1. Install Git](#1-install-git)
- [2. Configure Git](#2-configure-git)
- [3. Understanding Deploy Keys](#3-understanding-deploy-keys)
- [4. Generate Deploy Key](#4-generate-deploy-key)
- [5. Add Deploy Key to GitHub Repository](#5-add-deploy-key-to-github-repository)
- [6. Test Connection](#6-test-connection)
- [7. Clone Repository](#7-clone-repository)
- [8. Multiple Repositories Setup](#8-multiple-repositories-setup)
- [9. Deploy Keys in CI/CD Pipelines](#9-deploy-keys-in-cicd-pipelines)
- [10. Alternative: GitHub Apps](#10-alternative-github-apps)

---

## 1. Install Git

```bash
# Update packages
sudo apt update

# Install Git
sudo apt install git -y

# Verify installation
git --version
# Output: git version 2.x.x
```

---

## 2. Configure Git

```bash
# Set your identity
git config --global user.name "<your-name>"
git config --global user.email "<your-email>"

# Set default branch name
git config --global init.defaultBranch main

# Verify configuration
git config --list
```

---

## 3. Understanding Deploy Keys
https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys

### What are Deploy Keys?

Deploy keys are SSH keys that grant access to a **single repository**. Unlike personal SSH keys (added to your GitHub account), deploy keys are attached directly to a repository.

```
┌─────────────────────────────────────────────────────────────┐
│                    Personal SSH Key                          │
│     (Added to GitHub Account → Access to ALL your repos)     │
└─────────────────────────────────────────────────────────────┘
                              vs
┌─────────────────────────────────────────────────────────────┐
│                      Deploy Key                              │
│     (Added to Repository → Access to ONLY that repo)         │
└─────────────────────────────────────────────────────────────┘
```

### Why Deploy Keys for Servers?

| Aspect         | Personal SSH Key      | Deploy Key                   |
| -------------- | --------------------- | ---------------------------- |
| Scope          | All your repositories | Single repository            |
| If compromised | All repos at risk     | Only one repo at risk        |
| Tied to        | Your GitHub account   | The repository               |
| Best for       | Local development     | Servers, CI/CD               |
| Write access   | Always                | Optional (read-only default) |

---

## 4. Generate Deploy Key

### Generate a dedicated key for each repository:

```bash
# Create a descriptive filename for the key
# Format: deploy_key_<reponame>

ssh-keygen -t ed25519 -C "deploy-key-myapp" -f ~/.ssh/deploy_key_myapp
```

### Prompts:

```
Enter passphrase (empty for no passphrase):
# Press Enter (no passphrase for automation)
# Deploy keys typically don't use passphrases for CI/CD automation

Enter same passphrase again:
# Press Enter
```

### Verify key was created:

```bash
ls -la ~/.ssh/deploy_key_myapp*
# Should show:
# deploy_key_myapp      (private key)
# deploy_key_myapp.pub  (public key)
```

### Set correct permissions:

```bash
chmod 600 ~/.ssh/deploy_key_myapp
chmod 644 ~/.ssh/deploy_key_myapp.pub
```

```bash
# Private key must be 600
chmod 600 ~/.ssh/deploy_key_myapp

# Public key should be 644
chmod 644 ~/.ssh/deploy_key_myapp.pub

# .ssh directory must be 700
chmod 700 ~/.ssh
```

---

## 5. Add Deploy Key to GitHub Repository

### Step 5.1: Copy the public key

```bash
cat ~/.ssh/deploy_key_myapp.pub
```

### Step 5.2: Add to GitHub Repository

1. Go to your repository on **GitHub.com**
2. Click **Settings** (repository settings, not account settings)
3. In sidebar, click **Deploy keys**
4. Click **Add deploy key**
5. Fill in:
   - **Title**: Any name
   - **Key**: Paste the public key content
   - **Allow write access**: ☐ Check only if you need push access
6. Click **Add key**

### Write Access:

- **Unchecked (default)**: Read-only - can clone and pull
- **Checked**: Read/Write - can also push (needed if server pushes changes)

For most deployments, **read-only is sufficient** and more secure.

---

## 6. Test Connection

### Configure SSH to use the deploy key:

```bash
nano ~/.ssh/config
```

Add:

```
Host github.com-myapp
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_myapp
    IdentitiesOnly yes
```

### Test the connection:

```bash
ssh -T git@github.com-myapp
```

First time prompt:

```
The authenticity of host 'github.com (IP)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
Are you sure you want to continue connecting (yes/no)? yes
```

Successful output:

```
Hi username/myapp! You've successfully authenticated, but GitHub does not provide shell access.
```

Note: It shows `username/myapp` (repo) instead of just `username` (account) - this confirms deploy key is working!

---

## 7. Clone Repository

### Clone using the configured host alias:

```bash
# Use the host alias from SSH config
git clone git@github.com-myapp:username/myapp.git

# OR

ssh -i ~/.ssh/deploy_key_myapp git@github.com:username/myapp.git
```

### Important: Use the custom host!

```bash
# ✅ Correct - uses deploy key via alias
git clone git@github.com-myapp:username/myapp.git

# ❌ Wrong - tries to use default key
git clone git@github.com:username/myapp.git
```

### After cloning, verify remote:

```bash
cd myapp
git remote -v
# origin  git@github.com-myapp:username/myapp.git (fetch)
# origin  git@github.com-myapp:username/myapp.git (push)
```

### Pull updates:

```bash
git pull origin main
```

---

## 8. Multiple Repositories Setup

Since deploy keys can only be used for ONE repository, you need separate keys for each repo.

### Step 8.1: Generate keys for each repository

```bash
# Backend API
ssh-keygen -t ed25519 -C "deploy-backend-api" -f ~/.ssh/deploy_key_backend_api -N ""

# Frontend App
ssh-keygen -t ed25519 -C "deploy-frontend-app" -f ~/.ssh/deploy_key_frontend_app -N ""

# Shared Library
ssh-keygen -t ed25519 -C "deploy-shared-lib" -f ~/.ssh/deploy_key_shared_lib -N ""
```

> **Note**: `-N ""` sets empty passphrase non-interactively

### Step 8.2: Add each public key to respective repository

```bash
# Display each public key and add to corresponding repo
cat ~/.ssh/deploy_key_backend_api.pub    # Add to backend-api repo
cat ~/.ssh/deploy_key_frontend_app.pub   # Add to frontend-app repo
cat ~/.ssh/deploy_key_shared_lib.pub     # Add to shared-lib repo
```

### Step 8.3: Configure SSH for all repositories

```bash
nano ~/.ssh/config
```

```
# Backend API Repository
Host github.com-backend-api
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_backend_api
    IdentitiesOnly yes

# Frontend App Repository
Host github.com-frontend-app
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_frontend_app
    IdentitiesOnly yes

# Shared Library Repository
Host github.com-shared-lib
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_shared_lib
    IdentitiesOnly yes
```

### Step 8.4: Clone each repository

```bash
# Clone using respective aliases
git clone git@github.com-backend-api:myorg/backend-api.git
git clone git@github.com-frontend-app:myorg/frontend-app.git
git clone git@github.com-shared-lib:myorg/shared-lib.git
```

### Diagram: Multiple Repos Setup

```
~/.ssh/
├── config                      # SSH configuration
├── deploy_key_backend_api      # Private key for backend-api
├── deploy_key_backend_api.pub  # Public key → added to backend-api repo
├── deploy_key_frontend_app     # Private key for frontend-app
├── deploy_key_frontend_app.pub # Public key → added to frontend-app repo
├── deploy_key_shared_lib       # Private key for shared-lib
└── deploy_key_shared_lib.pub   # Public key → added to shared-lib repo
```

---

## 9. Deploy Keys in CI/CD Pipelines

Deploy keys are essential for CI/CD pipelines (like GitHub Actions) to clone private repositories.

### Overview: How it works in CI/CD

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Actions Workflow                   │
├─────────────────────────────────────────────────────────────┤
│  1. Workflow triggered (push/PR)                            │
│  2. Runner starts                                           │
│  3. SSH key loaded from GitHub Secrets                      │
│  4. Clone repository using deploy key                       │
│  5. Build, test, deploy                                     │
│  6. SSH to server (optional) or push Docker image           │
└─────────────────────────────────────────────────────────────┘
```

### Method 1: Store Deploy Key in GitHub Secrets

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Setup SSH Key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DEPLOY_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan github.com >> ~/.ssh/known_hosts

      - name: Clone Repository
        run: |
          GIT_SSH_COMMAND="ssh -i ~/.ssh/deploy_key" git clone git@github.com:user/repo.git
```

### Method 2: Using webfactory/ssh-agent action (Recommended)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Setup SSH Agent
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.DEPLOY_KEY }}

      - name: Clone Private Repo
        run: git clone git@github.com:user/private-repo.git
```

### Adding Deploy Key to GitHub Secrets:

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `DEPLOY_KEY`
4. Value: Paste the **private key** content (the one without `.pub`)
5. Click **Add secret**

```bash
# Get private key content
cat ~/.ssh/deploy_key_myapp
# Copy entire content including:
# -----BEGIN OPENSSH PRIVATE KEY-----
# ...
# -----END OPENSSH PRIVATE KEY-----
```

### For SSH into Server (Deployment):

You'll also need server SSH keys in CI/CD. This will be covered in detail in the CI/CD phase, but the concept is:

```yaml
- name: Deploy to Server
  run: |
    # Use a separate SSH key for server access
    echo "${{ secrets.SERVER_SSH_KEY }}" > ~/.ssh/server_key
    chmod 600 ~/.ssh/server_key
    ssh -i ~/.ssh/server_key deploy@your-server.com "cd /app && git pull && docker-compose up -d"
```

> 📝 **We'll create comprehensive CI/CD guides in Phase 3** covering GitHub Actions workflows for Django/FastAPI deployments.

---

## 10. Alternative: GitHub Apps

GitHub also recommends **GitHub Apps** for more complex scenarios. This is a more advanced approach with additional benefits.

### When to use GitHub Apps instead of Deploy Keys:

| Use Case                 | Deploy Keys                | GitHub Apps             |
| ------------------------ | -------------------------- | ----------------------- |
| Single repository        | ✅ Perfect                 | Overkill                |
| Multiple repositories    | 🔶 Multiple keys needed    | ✅ Single app           |
| Fine-grained permissions | ❌ Read or Read/Write only | ✅ Granular permissions |
| Token expiration         | ❌ Never expires           | ✅ 1-hour tokens        |
| Audit logging            | Basic                      | ✅ Detailed             |
| Rate limits              | Standard                   | ✅ Higher limits        |

### High-Level Overview of GitHub Apps:

1. **Create a GitHub App** in your organization/account settings
2. **Define permissions** (e.g., read-only content access)
3. **Generate a private key** for the app
4. **Install the app** on repositories that need access
5. **Generate installation tokens** (1-hour expiry) in your CI/CD pipeline
6. **Use tokens** for git operations

### Resources:

- [GitHub Apps Documentation 1](https://docs.github.com/en/apps/creating-github-apps)
- [GitHub Apps Documentation 2](https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/about-creating-github-apps)
- [GitHub Apps Documentation 3](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)

---

## 📌 Quick Reference

```bash
# 1. Generate deploy key (no passphrase for automation)
ssh-keygen -t ed25519 -C "deploy-myapp-prod" -f ~/.ssh/deploy_key_myapp -N ""

# 2. Add public key to GitHub repository
#    Repository → Settings → Deploy keys → Add deploy key
cat ~/.ssh/deploy_key_myapp.pub

# 3. Configure SSH
cat >> ~/.ssh/config << 'EOF'
Host github.com-myapp
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_myapp
    IdentitiesOnly yes
EOF

# 4. Test connection
ssh -T git@github.com-myapp

# 5. Clone repository
git clone git@github.com-myapp:youruser/myapp.git
```

---
