# Firewall Configuration with UFW

> **Goal**: Secure the server by allowing only necessary incoming traffic.

## Table of Contents
- [1. Understanding UFW](#1-understanding-ufw)
- [2. Install and Enable UFW](#2-install-and-enable-ufw)
- [3. Default Policies](#3-default-policies)
- [4. Allow Essential Services](#4-allow-essential-services)
- [5. Enable UFW](#5-enable-ufw)
- [6. Common Rules](#6-common-rules)
- [7. Managing Rules](#7-managing-rules)
- [8. Application Profiles](#8-application-profiles)
- [9. Common Deployment Scenarios](#9-common-deployment-scenarios)
- [10. Monitoring and Logs](#10-monitoring-and-logs)

---

## 1. Understanding UFW

**UFW (Uncomplicated Firewall)** is a frontend for iptables, designed to make firewall configuration easier.

### How it works:
```
Internet → Firewall (UFW) → Your Server Applications
                ↓
        Blocks unauthorized access
```

### What we'll allow:
| Port | Service | Required? |
|------|---------|-----------|
| 22 | SSH | ✅ Always |
| 80 | HTTP | ✅ For web apps |
| 443 | HTTPS | ✅ For secure web apps |

---

## 2. Install and Enable UFW

```bash
# UFW is usually pre-installed on Ubuntu
# If not:
sudo apt install ufw -y

# Check status
sudo ufw status
# Output: Status: inactive (if not enabled)
```

---

## 3. Default Policies

**Set secure defaults BEFORE enabling!**

```bash
# Deny all incoming connections by default
sudo ufw default deny incoming

# Allow all outgoing connections (your server can access internet)
sudo ufw default allow outgoing
```

### Why these defaults?
- **Deny incoming**: Block everything unless explicitly allowed
- **Allow outgoing**: Your apps need to access external APIs, download packages, etc.

---

## 4. Allow Essential Services

⚠️ **CRITICAL**: Allow SSH BEFORE enabling UFW or you'll be locked out!

```bash
# Allow SSH (MUST DO THIS FIRST!)
sudo ufw allow ssh
# or specifically:
sudo ufw allow 22/tcp

# If you're using a custom SSH port (e.g., 2222):
sudo ufw allow 2222/tcp
```

### Allow Web Traffic:
```bash
# HTTP
sudo ufw allow 80/tcp

# HTTPS
sudo ufw allow 443/tcp

# Or allow both at once:
sudo ufw allow http
sudo ufw allow https
```

---

## 5. Enable UFW

```bash
# Enable the firewall
sudo ufw enable
```

### Verify:
```bash
sudo ufw status verbose
```

Expected output:
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

---

## 6. Common Rules

### Allow specific IP addresses:
```bash
# Allow specific IP to access SSH
sudo ufw allow from 203.0.113.50 to any port 22

# Allow specific IP to access any port
sudo ufw allow from 203.0.113.50
```

### Allow port ranges:
```bash
# Allow ports 3000-3010 for development
sudo ufw allow 3000:3010/tcp
```

### Allow specific network interface:
```bash
# Allow on specific interface (useful for internal networks)
sudo ufw allow in on eth0 to any port 5432
```

### Deny specific IP:
```bash
# Block a malicious IP
sudo ufw deny from 203.0.113.100
```

---

## 7. Managing Rules

### View numbered rules (for deletion):
```bash
sudo ufw status numbered
```

Output:
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
```

### Delete a rule:
```bash
# By number
sudo ufw delete 3

# By rule specification
sudo ufw delete allow 80/tcp
```

### Reset all rules:
```bash
sudo ufw reset
```

### Disable UFW temporarily:
```bash
sudo ufw disable
```

---

## 8. Application Profiles

UFW has pre-defined application profiles.

### View available profiles:
```bash
sudo ufw app list
```

### View profile details:
```bash
sudo ufw app info "Nginx Full"
```

Output:
```
Profile: Nginx Full
Title: Web Server (Nginx, HTTP + HTTPS)
Description: Small, but very powerful and efficient web server

Ports:
  80,443/tcp
```

### Use application profile:
```bash
sudo ufw allow "Nginx Full"
sudo ufw allow "OpenSSH"
```

---

## 9. Common Deployment Scenarios

### Web Application (Production):
```bash
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

### Development Server:
```bash
sudo ufw allow 3000/tcp  # Node/React dev server
sudo ufw allow 8000/tcp  # Django dev server
sudo ufw allow 8080/tcp  # Generic dev port
sudo ufw enable
```

### Database Server (Internal Only):
```bash
# Allow PostgreSQL only from app server
sudo ufw allow from 10.0.0.5 to any port 5432
sudo ufw enable
```

---

## 10. Monitoring and Logs

### Check UFW logs:
```bash
# View UFW logs
sudo tail -f /var/log/ufw.log

# Enable/adjust logging level
sudo ufw logging on
sudo ufw logging medium  # low, medium, high, full
```

---

## ✅ Recommended Production Rules

```bash
# Reset and start fresh
sudo ufw reset

# Set defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Essential ports
sudo ufw allow ssh          # Port 22
sudo ufw allow http         # Port 80
sudo ufw allow https        # Port 443

# Enable
sudo ufw enable

# Verify
sudo ufw status verbose
```

---

## 📌 Quick Reference Commands

```bash
sudo ufw status              # Check status
sudo ufw status numbered     # List rules with numbers
sudo ufw enable              # Enable firewall
sudo ufw disable             # Disable firewall
sudo ufw allow 80/tcp        # Allow port
sudo ufw deny 80/tcp         # Deny port
sudo ufw delete allow 80/tcp # Remove rule
sudo ufw reset               # Reset all rules
sudo ufw reload              # Reload rules
```

---
