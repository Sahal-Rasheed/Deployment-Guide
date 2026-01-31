# Initial Server Setup - Ubuntu VPS

> **Goal**: Transform a fresh VPS into a secure, production-ready server.

## Table of Contents
- [1. First Connection to Server](#1-first-connection-to-server)
- [2. Update System Packages](#2-update-system-packages)
- [3. Create Non-Root User](#3-create-non-root-user)
- [4. Grant Sudo Privileges](#4-grant-sudo-privileges)
- [5. Set Up SSH Key Authentication](#5-set-up-ssh-key-authentication)
- [6. Disable Password Authentication](#6-disable-password-authentication)
- [7. Disable Root Login](#7-disable-root-login)
- [8. Configure Timezone (Optional)](#8-configure-timezone-optional)
- [9. Set Hostname (Optional)](#9-set-hostname-optional)

---

## 1. First Connection to Server

When you purchase a VPS, you typically receive:
- IP address
- Root password (via email or dashboard)
- SSH port (usually 22)

### Connect from your local machine:

```bash
# Basic SSH connection
ssh root@YOUR_SERVER_IP

# If using a different port
ssh -p PORT_NUMBER root@YOUR_SERVER_IP
```

---

## 2. Update System Packages

**Always update the system!**

```bash
# Update package list
apt update

# Upgrade installed packages
apt upgrade -y

# Remove unnecessary packages
apt autoremove -y

# (Optional) Upgrade distribution
apt dist-upgrade -y
```

---

## 3. Create Non-Root User

**Never run applications as root!** This is a security practice.

```bash
# Create a new user (replace 'deploy' with your preferred name)
adduser deploy
```

You'll be prompted for:
```
New password: [enter strong password]
Retype new password: [confirm password]
Full Name []: Deploy User
Room Number []: [press Enter]
Work Phone []: [press Enter]
Home Phone []: [press Enter]
Other []: [press Enter]
Is the information correct? [Y/n] Y
```

---

## 4. Grant Sudo Privileges

```bash
# Add user to sudo group
usermod -aG sudo deploy
```

### Verify sudo access:
```bash
# Switch to new user 
su - deploy      <OR>  su deploy  

# Test sudo (will ask for password)
sudo whoami
# Should output: root

# Exit back to root
exit
```

---

## 5. Set Up SSH Key Authentication

### Step 5.1: Generate SSH Key (On LOCAL Machine)

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**Prompts explained:**
```
Enter file in which to save the key (/home/youruser/.ssh/id_ed25519): 
# Press Enter for default, or specify custom path like:
# /home/youruser/.ssh/my_vps_key

Enter passphrase (empty for no passphrase): 
# Recommended: Add a passphrase for extra security
```

### Key Types:
| Type | Command | Security | Compatibility |
|------|---------|----------|---------------|
| ED25519 | `-t ed25519` | ⭐⭐⭐⭐⭐ | Modern systems |
| RSA 4096 | `-t rsa -b 4096` | ⭐⭐⭐⭐ | Maximum compatibility |

### Step 5.2: Copy Public Key to Server

**Method 1: Using ssh-copy-id (Easiest)**
```bash
# On LOCAL machine
ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@YOUR_SERVER_IP
```

**Method 2: Manual Copy**
```bash
# On LOCAL machine
cat ~/.ssh/id_ed25519.pub
# Copy the output
```

```bash
# On SERVER as the deploy user
su - deploy

# Create .ssh directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Create authorized_keys file and paste the copied key
nano ~/.ssh/authorized_keys

# Set correct permissions
chmod 600 ~/.ssh/authorized_keys
```

### Step 5.3: Test SSH Key Login

```bash
ssh deploy@YOUR_SERVER_IP

# Should login without password (or with key passphrase only)
```

⚠️ **IMPORTANT**: Test ssh key login BEFORE disabling password auth!

---

## 6. Disable Password Authentication

**Only do this AFTER confirming SSH key login works!**

```bash
# On SERVER
sudo nano /etc/ssh/sshd_config
```

### Find and modify these lines:

```bash
# Change these settings:
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM no
```

### Apply changes:
```bash
# Test configuration for syntax errors
sudo sshd -t

# If no errors, restart SSH service
sudo systemctl restart sshd
```

### Verify (from new terminal on LOCAL machine):
```bash
# This should fail
ssh -o PubkeyAuthentication=no deploy@YOUR_SERVER_IP
# Expected: Permission denied (publickey)

# This should work
ssh deploy@YOUR_SERVER_IP
```

---

## 7. Disable Root Login

```bash
sudo nano /etc/ssh/sshd_config
```

### Find and modify:
```bash
PermitRootLogin no
```

### Restart SSH:
```bash
sudo systemctl restart sshd
```

---

## 8. Configure Timezone (Optional)

```bash
# Check current timezone
timedatectl

# List available timezones
timedatectl list-timezones

# Set timezone (example: Asia/Kolkata for India)
sudo timedatectl set-timezone Asia/Kolkata

# Verify
date
```

---

## 9. Set Hostname (Optional)

```bash
# Set hostname
sudo hostnamectl set-hostname your-server-name

# Update hosts file
sudo nano /etc/hosts
```

Add this line:
```
127.0.1.1   your-server-name
```

### Verify:
```bash
hostname
hostnamectl
```

---

## 📌 Quick Reference Commands

```bash
# Check Ubuntu version
lsb_release -a

# Check system resources
htop     # or top
free -h  # memory
df -h    # disk space

# Check running services
systemctl list-units --type=service --state=running

# View system logs
journalctl -xe
```

---
