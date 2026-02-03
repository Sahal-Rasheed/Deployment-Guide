# Domain, DNS & SSL Configuration

> **Goal**: Purchase a domain, configure DNS, and set up SSL certificates with auto-renewal for production deployments including subdomain-based architecture.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Domain Purchase](#2-domain-purchase)
- [3. DNS Concepts](#3-dns-concepts)
- [4. Cloudflare DNS Setup](#4-cloudflare-dns-setup)
- [5. Subdomain Architecture](#5-subdomain-architecture)
- [6. SSL/TLS Concepts](#6-ssltls-concepts)
- [7. Certbot Installation](#7-certbot-installation)
- [8. Single Domain SSL](#8-single-domain-ssl)
- [9. Multiple Subdomains SSL](#9-multiple-subdomains-ssl)
- [10. Wildcard SSL Certificate](#10-wildcard-ssl-certificate)
- [11. Auto-Renewal Setup](#11-auto-renewal-setup)
- [12. Nginx SSL Configuration](#12-nginx-ssl-configuration)
- [13. Troubleshooting](#13-troubleshooting)

---

## 1. Overview

### What we're setting up:

```
┌─────────────────────────────────────────────────────────────┐
│                    YOUR DOMAIN                               │
│                   example.com                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   │ example.com  │  │api.example.  │  │app.example.  │      │
│   │   (Landing)  │  │    com       │  │    com       │      │
│   │              │  │  (Backend)   │  │  (Frontend)  │      │
│   └──────────────┘  └──────────────┘  └──────────────┘      │
│          │                 │                 │               │
│          └─────────────────┼─────────────────┘               │
│                            ▼                                 │
│              ┌─────────────────────────┐                    │
│              │     YOUR VPS SERVER     │                    │
│              │      (Single IP)        │                    │
│              │   Nginx routes traffic  │                    │
│              └─────────────────────────┘                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### The Complete Flow:

```
1. User types: https://api.example.com
                    │
                    ▼
2. DNS Query: "What's the IP of api.example.com?"
                    │
                    ▼
3. Cloudflare DNS: "It's 123.45.67.89"
                    │
                    ▼
4. Browser connects to 123.45.67.89:443 (HTTPS)
                    │
                    ▼
5. Nginx receives request, checks server_name
                    │
                    ▼
6. SSL handshake using Let's Encrypt certificate
                    │
                    ▼
7. Request proxied to backend application
```

---

## 2. Domain Purchase

### Where to Buy:

You can purchase domains from any registrar:

| Registrar      | Pros                                 
| -------------- | ------------------------------------
| **Cloudflare** | At-cost pricing, free DNS, no markup 
| Namecheap      | Good prices, free WhoisGuard         
| Google Domains | Clean interface, good DNS            
| GoDaddy        | Popular, but upsells aggressively    
| Porkbun        | Cheap, good interface                

> In this guide we are going to use Cloudflare for Domain & DNS management. You can buy your domain anywhere and point nameservers to Cloudflare if you prefer DNS management using Cloudflare.

### Buying from Cloudflare:

1. Create account at [cloudflare.com](https://cloudflare.com)
2. Go to **Registrar** → **Register Domains**
3. Search for your domain
4. Complete purchase (requires payment method)
5. Domain is automatically configured with Cloudflare DNS

### Using Domain from Other Registrar:

If you bought elsewhere, point nameservers to Cloudflare:

1. Add site to Cloudflare (free plan)
2. Cloudflare provides nameservers:
   ```
   ns1.cloudflare.com
   ns2.cloudflare.com
   ```
3. Go to your registrar → Domain Settings → Nameservers
4. Replace default nameservers with Cloudflare's
5. Wait 24-48 hours for propagation (usually faster)

---

## 3. DNS Concepts

### What is DNS?

DNS (Domain Name System) translates human-readable domain names to IP addresses.

```
example.com  →  DNS  →  123.45.67.89
```

### DNS Record Types:

https://www.cloudflare.com/en-gb/learning/dns/dns-records/

| Type      | Purpose                          | Example                            |
| --------- | -------------------------------- | ---------------------------------- |
| **A**     | Points domain to IPv4 address    | `example.com → 123.45.67.89`       |
| **AAAA**  | Points domain to IPv6 address    | `example.com → 2001:db8::1`        |
| **CNAME** | Forwards one domain or subdomain to another domain, does NOT provide an IP address. Used for: Subdomains, cloud services, CDNs          | `www.example.com → example.com`    |
| **MX**    | Mail server for domain, Used for: Email routing           | `example.com → mail.example.com`   |
| **TXT**   | Text records (verification, SPF) - Stores arbitrary text data | `example.com → "v=spf1..."`        |
| **NS**    | Stores the name server for a DNS entry            | `example.com → ns1.cloudflare.com` |

### A Record vs CNAME:

```
A Record:
┌─────────────────┐     ┌─────────────────┐
│ api.example.com │ ──▶ │   123.45.67.89  │  (Direct IP)
└─────────────────┘     └─────────────────┘

CNAME Record:
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ www.example.com │ ──▶ │   example.com   │ ──▶ │   123.45.67.89  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                         (Alias first)          (Then resolves IP)
```

### TTL (Time To Live):

TTL defines how long DNS records are cached:

| TTL Value     | Use Case                               |
| ------------- | -------------------------------------- |
| 300 (5 min)   | During setup/testing, frequent changes |
| 3600 (1 hour) | Normal operation                       |
| 86400 (1 day) | Stable, rarely changing records        |

> **Tip**: Set low TTL (300) when making changes, increase after stable.

---

## 4. Cloudflare DNS Setup

### Step 4.1: Add Your Site to Cloudflare

1. Login to [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Click **Add a Site**
3. Enter your domain: `example.com`
4. Select **Free** plan
5. Cloudflare scans existing DNS records

### Step 4.2: Configure DNS Records

Navigate to **DNS** → **Records** → **Add record**

#### For Root Domain (@):

```
Type: A
Name: @                    (or example.com)
IPv4: YOUR_SERVER_IP       (e.g., 123.45.67.89)
Proxy: DNS only (gray)     (for SSL setup, disable proxy initially)
TTL: Auto
```

#### For www subdomain:

```
Type: CNAME
Name: www
Target: example.com
Proxy: DNS only (gray)
TTL: Auto
```

#### For API subdomain:

```
Type: A
Name: api
IPv4: YOUR_SERVER_IP
Proxy: DNS only (gray)
TTL: Auto
```

#### For App/Frontend subdomain:

```
Type: A
Name: app
IPv4: YOUR_SERVER_IP
Proxy: DNS only (gray)
TTL: Auto
```

### Step 4.3: Proxy Status Explained

Cloudflare offers two modes:

| Mode         | Icon            | Traffic Flow                    | When to Use                                  |
| ------------ | --------------- | ------------------------------- | -------------------------------------------- |
| **Proxied**  | 🟠 Orange cloud | User → Cloudflare → Your Server | After SSL is set up, for CDN/DDoS protection |
| **DNS only** | ⚪ Gray cloud   | User → Your Server (direct)     | During SSL setup with Certbot                |

> ⚠️ **Important**: Keep **DNS only** (gray cloud) during initial Certbot SSL setup. You can enable proxy later. 

> ⚠️ **Note**: **Nginx Reverse Proxy vs Cloudflare Proxy** -> Even if **Nginx** is already configured as a reverse proxy, enabling **Cloudflare** proxy is still recommended. Nginx operates at the **server level (routing, app logic, SSL termination)**. Cloudflare operates at the **internet edge (DDoS protection, CDN, IP masking)**. Cloudflare stops malicious or excessive traffic before it reaches the server, something Nginx alone cannot do. This “double proxy” setup **(Cloudflare → Nginx → App)** is a standard, production-grade architecture and is supported on Cloudflare’s free plan.

> ⚠️ During initial SSL setup with Certbot, keep DNS records set to DNS only (gray cloud). This is because Certbot needs to directly reach your server to complete the HTTP-01 challenge, **if Cloudflare proxy is enabled, it may interfere with this process**. Once SSL is working, switch to Proxied (orange cloud) and set Cloudflare SSL mode to Full (Strict) based on your needs.

### Step 4.4: Verify DNS Propagation

```bash
# Check DNS resolution
dig example.com +short
# Should return: YOUR_SERVER_IP

dig api.example.com +short
# Should return: YOUR_SERVER_IP

# Or use nslookup
nslookup example.com
nslookup api.example.com

# Online tools:
# https://dnschecker.org
# https://www.whatsmydns.net
```

---

## 5. Subdomain Architecture

### Why Use Subdomains?

Subdomain-based architecture separates concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                    example.com                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  example.com        → Landing page / Marketing site          │
│  app.example.com    → Frontend application (React/Vue)       │
│  api.example.com    → Backend API (Django/FastAPI)           │
│  admin.example.com  → Admin panel                            │
│  docs.example.com   → Documentation                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Benefits:

| Benefit                 | Description                                        |
| ----------------------- | -------------------------------------------------- |
| **Separation**          | Frontend and Backend are clearly separated         |
| **Independent Scaling** | Can move services to different servers later       |
| **CORS Simplicity**     | Clear origin boundaries                            |
| **SSL Flexibility**     | Can use wildcard or individual certs               |
| **Clear URLs**          | `api.example.com/users` vs `example.com/api/users` |

### Our Architecture:

For this guide, we'll set up:

```
example.com       →  Landing/Marketing (optional)
app.example.com   →  Frontend (React/Vue) - Port 3000
api.example.com   →  Backend (Django/FastAPI) - Port 8000
```

### DNS Records for Subdomains:

All subdomains point to the same server IP. Nginx handles routing based on `server_name`.

```
# In Cloudflare DNS: (Create A Records for each subdomain)

Type    Name    Content           Proxy
─────────────────────────────────────────
A       @       YOUR_SERVER_IP    DNS only (root domain)
A       app     YOUR_SERVER_IP    DNS only
A       api     YOUR_SERVER_IP    DNS only
A       www     YOUR_SERVER_IP    DNS only
```

---

## 6. SSL/TLS Concepts

### What is SSL/TLS?

SSL/TLS encrypts traffic between the user's browser and your server.

```
Without SSL (HTTP):
Browser ──────── Plain text ────────▶ Server
         (Anyone can read/modify)

With SSL (HTTPS):
Browser ──────── Encrypted ────────▶ Server
         (Only browser & server can read)
```

### Let's Encrypt:

[Let's Encrypt](https://letsencrypt.org) is a free, automated Certificate Authority (CA).

| Feature        | Description                   |
| -------------- | ----------------------------- |
| **Price**      | Free                          |
| **Validity**   | 90 days (auto-renewable)      |
| **Trust**      | Trusted by all major browsers |
| **Automation** | Certbot handles everything    |

### Certificate Types for Subdomains:

| Type                 | Covers                                              | Best For                     |
| -------------------- | --------------------------------------------------- | ---------------------------- |
| **Single Domain**    | `example.com` only                                  | Simple sites                 |
| **SAN/Multi-domain** | `example.com`, `api.example.com`, `app.example.com` | Few specific subdomains      |
| **Wildcard**         | `*.example.com` (all subdomains)                    | Many subdomains, flexibility |

### Challenge Types:

Certbot must prove you control the domain. Two methods:

| Challenge   | How it Works                                  | When to Use                          |
| ----------- | --------------------------------------------- | ------------------------------------ |
| **HTTP-01** | Places file at `/.well-known/acme-challenge/` | Single domains, simple setup         |
| **DNS-01**  | Creates TXT record in DNS                     | Wildcard certs, no web server needed |

> ⚠️ **Important**: Wildcard certificates (`*.example.com`) **require DNS-01 challenge**.

---

## 7. Certbot Installation

### Installation (Recommended):

Certbot can be installed via **Snap or APT**.
Although Snap is officially recommended in the Certbot documentation, it can introduce issues on some servers (sandboxing limitations, path resolution problems, service reload failures, and incompatibility with minimal images).

For production servers, APT-based installation is preferred due to better compatibility, predictable behavior, and easier debugging.

Official guide at [certbot.eff.org](https://certbot.eff.org/)

```bash
sudo apt update

# Install Certbot & Nginx plugin via apt
sudo apt install certbot python3-certbot-nginx -y

# Verify installation
certbot --version
# Output: certbot 2.x.x
```

### Install Cloudflare DNS Plugin (for Wildcard Certs):

```bash
# Required for DNS-01 challenge with Cloudflare
sudo apt install python3-certbot-dns-cloudflare -y
```

---

## 8. Single Domain SSL

### For a single domain without subdomains:

```bash
# Get certificate and auto-configure Nginx
sudo certbot --nginx -d example.com -d www.example.com
```

### Prompts:

```
Enter email address: your@email.com
Agree to terms: Y
Share email with EFF: N (optional)
```

> Note: You can also run `sudo certbot --nginx` and follow interactive prompts to add domains. As it will list all `server_names` found in Nginx configs. By using `-d` flags, you explicitly specify which domains to include.

### What happens:

1. Certbot verifies domain ownership (HTTP-01 challenge)
2. Obtains certificate from Let's Encrypt
3. Modifies Nginx config to use SSL
4. Sets up HTTP → HTTPS redirect

### Verify:

```bash
# Check certificate
sudo certbot certificates

# Test HTTPS
curl -I https://example.com
```

---

## 9. Multiple Subdomains SSL

### Option A: List All Subdomains (Recommended for few subdomains)

```bash
# Get certificate for multiple specific subdomains
sudo certbot --nginx \
  -d example.com \
  -d www.example.com \
  -d api.example.com \
  -d app.example.com
```

This creates ONE certificate covering all listed domains (SAN certificate).

### When You Add a New Subdomain:

> ⚠️ **Important**: When adding a new subdomain, you need to **expand** the existing certificate.

```bash
# Add a new subdomain (e.g., admin.example.com)
sudo certbot --nginx \
  --expand \
  -d example.com \
  -d www.example.com \
  -d api.example.com \
  -d app.example.com \
  -d admin.example.com
```

The `--expand` flag tells Certbot to add to existing certificate.

### Alternative: Separate Certificates

You can also get separate certificates for each subdomain:

```bash
# Separate certs (more management overhead)
sudo certbot --nginx -d example.com -d www.example.com
sudo certbot --nginx -d api.example.com
sudo certbot --nginx -d app.example.com
```

> **Note**: Each certificate renews independently. More to manage, but isolated.

---

## 10. Wildcard SSL Certificate

### Why Wildcard?

A wildcard certificate (`*.example.com`) covers ALL subdomains automatically:

- ✅ `api.example.com`
- ✅ `app.example.com`
- ✅ `admin.example.com`
- ✅ `anything.example.com`
- ❌ `example.com` (root domain needs separate entry)
- ❌ `sub.sub.example.com` (only one level)

### Wildcard Requires DNS-01 Challenge

Wildcard certs can't use HTTP-01 challenge. We must use DNS-01 with Cloudflare API.

### Step 10.1: Get Cloudflare API Token

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Click your profile (top right) → **My Profile**
3. Go to **API Tokens** → **Create Token**
4. Use template: **Edit zone DNS**
5. Configure permissions:

   ```
   Permissions:
   - Zone → DNS → Edit

   Zone Resources:
   - Include → Specific zone → example.com
   ```

6. Click **Continue to summary** → **Create Token**
7. **Copy the token immediately** (shown only once!)

### Step 10.2: Create Cloudflare Credentials File

```bash
# Create credentials directory
sudo mkdir -p /etc/letsencrypt/cloudflare

# Create credentials file
sudo nano /etc/letsencrypt/cloudflare/credentials.ini
```

Add your API token:

```ini
# Cloudflare API token
dns_cloudflare_api_token = YOUR_API_TOKEN_HERE
```

Secure the file:

```bash
# Restrict permissions (important!)
sudo chmod 600 /etc/letsencrypt/cloudflare/credentials.ini
```

### Step 10.3: Obtain Wildcard Certificate

```bash
# Get wildcard certificate for *.example.com AND example.com
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  -d example.com \
  -d "*.example.com" \
  --preferred-challenges dns-01
```

### Prompts:

```
Enter email address: your@email.com
Agree to terms: Y
Share email with EFF: N
```

### What happens:

1. Certbot creates DNS TXT record via Cloudflare API
2. Let's Encrypt verifies the TXT record
3. Certificate is issued
4. TXT record is automatically cleaned up

### Step 10.4: Verify Certificate

```bash
# List certificates
sudo certbot certificates

# Output:
# Certificate Name: example.com
# Domains: example.com *.example.com
# Expiry Date: 2026-05-03
```

### Step 10.5: Configure Nginx to Use Wildcard Cert

Since we used `certonly`, we need to manually configure Nginx:

```bash
# Edit your Nginx site config
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

    # SSL certificate (wildcard)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # SSL settings
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Your application config
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> **Note**: All subdomains use the SAME certificate path (`/etc/letsencrypt/live/example.com/`).

### Adding New Subdomains with Wildcard:

The beauty of wildcard certificates - **no certificate changes needed!**

1. Add DNS record in Cloudflare
2. Create Nginx server block
3. Use the same wildcard certificate
4. Reload Nginx

```bash
# Example: Adding admin.example.com

# 1. Add A record in Cloudflare DNS for 'admin'

# 2. Create Nginx config
sudo nano /etc/nginx/sites-available/admin.example.com
```

```nginx
server {
    listen 80;
    server_name admin.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name admin.example.com;

    # Same wildcard certificate!
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # ... rest of config
}
```

```bash
# 3. Enable and reload
sudo ln -s /etc/nginx/sites-available/admin.example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

**No certificate renewal or expansion needed!** 🎉

---

## 11. Auto-Renewal Setup

### How Auto-Renewal Works:

Let's Encrypt certificates expire after **90 days**. Certbot handles automatic renewal.

```
┌─────────────────────────────────────────────────────────────┐
│                    Auto-Renewal Process                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Day 1          Day 60         Day 90                        │
│    │              │              │                           │
│    ▼              ▼              ▼                           │
│  [Issued]    [Renewed]     [Expires]                        │
│              (30 days                                        │
│               before)                                        │
│                                                              │
│  Certbot runs twice daily via systemd timer                 │
│  Only renews if < 30 days remaining                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Verify Auto-Renewal is Configured:

```bash
# Check certbot timer status
sudo systemctl status certbot.timer

# Output should show:
# Active: active (waiting)
# Trigger: [next run time]
```

```bash
# List all timers
sudo systemctl list-timers | grep certbot

# Output:
# NEXT                        LEFT     LAST                        PASSED    UNIT
# Mon 2026-02-03 12:00:00 UTC 6h left  Mon 2026-02-03 00:00:00 UTC 6h ago    certbot.timer
```

### Test Renewal (Dry Run):

```bash
# Test renewal without actually renewing
sudo certbot renew --dry-run

# Expected output:
# Congratulations, all simulated renewals succeeded
```

### Renewal Hooks (Reload Nginx After Renewal):

Certbot needs to reload Nginx after renewing certificates:

```bash
# Create renewal hook
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

```bash
#!/bin/bash
# Reload Nginx after certificate renewal
systemctl reload nginx
```

```bash
# Make executable
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

### Manual Renewal (If Needed):

```bash
# Renew all certificates
sudo certbot renew

# Renew specific certificate
sudo certbot renew --cert-name example.com

# Force renewal (even if not due)
sudo certbot renew --force-renewal --cert-name example.com
```

### Renewal for Wildcard Certificates:

Wildcard certificates using DNS-01 challenge renew automatically as long as:

- Cloudflare credentials file exists and is valid
- API token has proper permissions
- DNS plugin is installed

```bash
# Test wildcard renewal
sudo certbot renew --dry-run --cert-name example.com
```

---

## 12. Nginx SSL Configuration

### Recommended SSL Settings:

Create a reusable SSL snippet:

```bash
sudo nano /etc/nginx/snippets/ssl-params.conf
```

```nginx
# SSL Parameters - Modern Configuration
# Reference: https://ssl-config.mozilla.org/

ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

# Modern protocols only
ssl_protocols TLSv1.2 TLSv1.3;

# Strong ciphers
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;

# HSTS (comment out if not ready for HSTS)
add_header Strict-Transport-Security "max-age=63072000" always;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 1.0.0.1 valid=300s;
resolver_timeout 5s;
```

### Complete Nginx Config for Subdomain:

```bash
sudo nano /etc/nginx/sites-available/api.example.com
```

```nginx
# HTTP - Redirect to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name api.example.com;

    # Redirect all HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

# HTTPS - Main Configuration
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name api.example.com;

    # SSL Certificate (wildcard or specific)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Include SSL parameters
    include snippets/ssl-params.conf;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Logging
    access_log /var/log/nginx/api.example.com.access.log;
    error_log /var/log/nginx/api.example.com.error.log;

    # Proxy to backend
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### Enable the Site:

```bash
# Create symlink
sudo ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Test SSL Configuration:

```bash
# Test with curl
curl -I https://api.example.com

# Check SSL certificate
echo | openssl s_client -servername api.example.com -connect api.example.com:443 2>/dev/null | openssl x509 -noout -dates

# Online SSL test (comprehensive)
# https://www.ssllabs.com/ssltest/
```

---

## 13. Troubleshooting

### DNS Issues:

```bash
# Check DNS resolution
dig api.example.com +short
dig api.example.com A

# If not resolving, check:
# 1. DNS record exists in Cloudflare
# 2. Propagation complete (can take up to 48h, usually minutes)
# 3. Correct server IP

# Clear local DNS cache
sudo systemd-resolve --flush-caches
```

### Certificate Issues:

```bash
# List all certificates
sudo certbot certificates

# Check certificate details
sudo openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -noout -text

# Check expiry
sudo openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -noout -enddate

# View certificate chain
sudo openssl s_client -connect api.example.com:443 -servername api.example.com
```

### Certbot Errors:

**"Challenge failed"**

```bash
# For HTTP-01 challenge:
# 1. Ensure Cloudflare proxy is OFF (gray cloud)
# 2. Ensure port 80 is open in firewall
# 3. Nginx is running and serving the domain

# Check if .well-known is accessible
curl http://example.com/.well-known/acme-challenge/test
```

**"DNS problem: NXDOMAIN"**

```bash
# Domain/subdomain doesn't exist in DNS
# Add the DNS record in Cloudflare and wait for propagation
```

**"Too many certificates already issued"**

```bash
# Let's Encrypt rate limits: 50 certs per week per domain
# Wait or use staging for testing:
sudo certbot --nginx -d example.com --staging
```

### Nginx SSL Errors:

**"SSL_CTX_use_certificate_chain_file failed"**

```bash
# Certificate file path is wrong or file doesn't exist
ls -la /etc/letsencrypt/live/example.com/

# Check permissions
sudo chmod 755 /etc/letsencrypt/live/
sudo chmod 755 /etc/letsencrypt/archive/
```

**"ssl_certificate_key does not match certificate"**

```bash
# Key and cert mismatch - regenerate certificate
sudo certbot delete --cert-name example.com
sudo certbot --nginx -d example.com
```

### Renewal Issues:

```bash
# Check renewal configuration
sudo cat /etc/letsencrypt/renewal/example.com.conf

# Test renewal
sudo certbot renew --dry-run

# Check certbot logs
sudo journalctl -u certbot
sudo cat /var/log/letsencrypt/letsencrypt.log
```

---

## 📌 Quick Reference

### DNS Setup Checklist:

```bash
# 1. Add DNS records in Cloudflare (all point to same IP)
@       A       YOUR_SERVER_IP    (DNS only)
www     A       YOUR_SERVER_IP    (DNS only)
api     A       YOUR_SERVER_IP    (DNS only)
app     A       YOUR_SERVER_IP    (DNS only)

# 2. Verify propagation
dig example.com +short
dig api.example.com +short
```

### SSL Commands:

```bash
# Install Certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Single/Multiple domains (HTTP-01)
sudo certbot --nginx -d example.com -d api.example.com -d app.example.com

# Wildcard certificate (DNS-01 with Cloudflare)
sudo snap install certbot-dns-cloudflare
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare/credentials.ini \
  -d example.com \
  -d "*.example.com"

# Expand existing certificate
sudo certbot --nginx --expand -d example.com -d api.example.com -d NEW.example.com

# List certificates
sudo certbot certificates

# Test renewal
sudo certbot renew --dry-run

# Force renewal
sudo certbot renew --force-renewal --cert-name example.com
```

### Adding New Subdomain Checklist:

**With Wildcard Certificate:**

```bash
# 1. Add DNS record in Cloudflare
# 2. Create Nginx config
sudo nano /etc/nginx/sites-available/new.example.com
# 3. Use same certificate path
# 4. Enable site
sudo ln -s /etc/nginx/sites-available/new.example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

**Without Wildcard (Expand Certificate):**

```bash
# 1. Add DNS record in Cloudflare
# 2. Create Nginx config (without SSL first)
# 3. Expand certificate
sudo certbot --nginx --expand -d example.com -d api.example.com -d app.example.com -d NEW.example.com
# 4. Reload Nginx
sudo nginx -t && sudo systemctl reload nginx
```

---

**Related Guide**: [Nginx Configuration →](./05-nginx.md)
