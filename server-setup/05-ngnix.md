# Nginx - Complete Guide

> **Goal**: Install, configure, and optimize Nginx as a reverse proxy for production deployments. Reference: https://www.freecodecamp.org/news/the-nginx-handbook/

## Table of Contents
- [1. Understanding Nginx](#1-understanding-nginx)
- [2. Installation](#2-installation)
- [3. File Structure](#3-file-structure)
- [4. Basic Configuration](#4-basic-configuration)
- [5. Server Blocks (Virtual Hosts)](#5-server-blocks-virtual-hosts)
- [6. Reverse Proxy Setup](#6-reverse-proxy-setup)
- [7. SSL/TLS with Let's Encrypt](#7-ssltls-with-lets-encrypt)
- [8. Security Headers](#8-security-headers)
- [9. Performance Optimization](#9-performance-optimization)
- [10. Load Balancing](#10-load-balancing)
- [11. Common Configurations](#11-common-configurations)
- [12. Logging](#12-logging)

---

## 1. Understanding Nginx

### What is Nginx?
Nginx is a high-performance web server, reverse proxy, and load balancer.

### Why use Nginx?

```
┌─────────────────────────────────────────────────────────────┐
│                        INTERNET                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      NGINX (Port 80/443)                     │
│  • SSL Termination       • Static Files                     │
│  • Reverse Proxy         • Load Balancing                   │
│  • Security Headers      • Rate Limiting                    │
│  • Caching               • Compression                      │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌──────────┐        ┌──────────┐        ┌──────────┐
    │ Django   │        │ FastAPI  │        │ React    │
    │ :8000    │        │ :8001    │        │ Static   │
    └──────────┘        └──────────┘        └──────────┘
```

### Key Benefits:
| Feature | Benefit |
|---------|---------|
| Reverse Proxy | Hide backend servers, single entry point |
| SSL Termination | Handle HTTPS at Nginx, HTTP to backends |
| Static Files | Serve static content efficiently |
| Load Balancing | Distribute traffic across servers |
| Security | Hide backend details, add security headers |

---

## 2. Installation

### Install Nginx:

```bash
# Update packages
sudo apt update

# Install Nginx
sudo apt install nginx -y

# Verify installation
nginx -v
# Output: nginx version: nginx/1.x.x
```

### Start and enable Nginx:

```bash
# Once the installation is done, NGINX should be automatically registered as a systemd service and should be running. To check, execute the following command:
sudo systemctl status nginx

# Start Nginx, if not running
sudo systemctl start nginx

# Enable on boot, if not already enabled
sudo systemctl enable nginx
```

### Verify it's working:

```bash
# Check if Nginx is listening
sudo ss -tlnp | grep nginx
# Should show :80

# Test with curl
curl http://localhost
# Should return Nginx welcome page HTML
```

### Allow through firewall:

```bash
# If using UFW
sudo ufw allow 'Nginx Full'

# Or individually:
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Visit `http://YOUR_SERVER_IP` in browser - you should see the Nginx welcome page!

---

## 3. File Structure

```
/etc/nginx/
├── nginx.conf              # Main configuration file
├── sites-available/        # Available site configurations
│   └── default             # Default site config (can add more here)
├── sites-enabled/          # Enabled sites (symlinks to sites-available)
│   └── default -> ../sites-available/default
├── conf.d/                 # Additional configuration files
├── snippets/               # Reusable configuration snippets
│   ├── fastcgi-php.conf
│   └── self-signed.conf
├── modules-available/      # Available modules
├── modules-enabled/        # Enabled modules
├── mime.types              # MIME type mappings (types)
└── fastcgi_params          # FastCGI parameters

/var/log/nginx/
├── access.log              # Access logs
└── error.log               # Error logs

/var/www/
└── html/                   # Default web root
    └── index.nginx-debian.html (default welcome page, site available)
```

### Key files explained:
- **nginx.conf**: Global settings, worker processes, includes
- **sites-available/**: Store all your site configs here
- **sites-enabled/**: Symlink configs to enable them
- **snippets/**: Reusable config blocks (SSL settings, etc.)

---

## 4. Basic Configuration

### Understand Directives and Contexts in NGINX

**Directives** are configuration statements that define how NGINX behaves. There are simple directives (single line) and block directives (enclose other directives).
**Simple Directive** consists of the directive name and the space delimited parameters, like `listen`, `return` and others. Simple directives are terminated by semicolons.
**Block Directive** are similar to simple directives, except that instead of ending with semicolons, they end with a pair of curly braces { } enclosing additional instructions. Examples include `http`, `server`, and `location` blocks. A block directive capable of containing other directives inside it is called a **context**, that is `events`, `http` and so on.

**Contexts** [Four core contexts in NGINX]:

`events { }` – The events context is used for setting global configuration regarding how NGINX is going to handle requests on a general level. There can be only one events context in a valid configuration file.

`http { }` – Evident by the name, http context is used for defining configuration regarding how the server is going to handle HTTP and HTTPS requests, specifically. There can be only one http context in a valid configuration file.

`server { }` – The server context is nested inside the http context and used for configuring specific virtual servers within a single host. There can be multiple server contexts in a valid configuration file nested inside the http context. Each server context is considered a virtual host.

`main` – The main context is the configuration file itself. Anything written outside of the three previously mentioned contexts is on the main context.

### Main nginx config file (`/etc/nginx/nginx.conf`):

```bash
sudo nano /etc/nginx/nginx.conf
```

```nginx
user www-data;          # current nginx user - www-data
worker_processes auto;  # Auto-detect CPU cores
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;  # Max connections per worker, you can check via `ulimit -n`
    # multi_accept on;
}

http {
    ##
    # Basic Settings
    ##

    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    # server_tokens off;

    # server_names_hash_bucket_size 64;
    # server_name_in_redirect off;

    # MIME Types - This auto includes the mime.types file that maps file extensions to MIME types, Thus we dont need to define them manually via types {} for our server configs Ngnix already knows them and can serve them correctly via this file
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ##
    # SSL Settings
    ##

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
    ssl_prefer_server_ciphers on;

    ##
    # Logging Settings
    ##

    access_log /var/log/nginx/access.log; # default access log location
    error_log /var/log/nginx/error.log;   # default error log location

    ##
    # Gzip Settings
    ##

    gzip on;

    # gzip_vary on;
    # gzip_proxied any;
    # gzip_comp_level 6;
    # gzip_buffers 16 8k;
    # gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    ##
    # Virtual Host Configs
    ##

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

> The above is the default nginx configuration file which will get autogenerated. Have added some comments added for clarity. You can modify settings like `worker_processes`, `worker_connections`, logging paths, gzip settings, etc. as per your requirements. Its recommended to keep most settings as default unless you know what you are doing. You should not directly add server {} for your sites here, instead use `sites-available/` dir for adding new site configs (site config file should only have server {} block) and symlink them to `sites-enabled/` to enable them. The `include /etc/nginx/sites-enabled/*;` line at the end of http {} block takes care of loading those site configs server block inside the http block.

### Test configuration & reload Nginx:

```bash
# Always test before reloading!
sudo nginx -t
# Output: nginx: configuration file /etc/nginx/nginx.conf test is successful

# Reload Nginx
sudo systemctl reload nginx # option 1

sudo nginx -s reload # option 2 (better)
```

---

## 5. Server Blocks (Virtual Hosts)

Server blocks allow hosting multiple sites on one server.

### Example: Serve a static site (myapp.com)

```bash
sudo mkdir -p /var/www/myapp.com/html
sudo chown -R $USER:$USER /var/www/myapp.com/html
echo "<h1>Hello from myapp.com</h1>" > /var/www/myapp.com/html/index.html
```

### Create a new site config:

```bash
# site config file for myapp.com (example)
sudo nano /etc/nginx/sites-available/myapp.com.conf
```

### Basic server block: (Serving static files)
Try this minimal config for serving static site:
```nginx
server {
    # listen on port 80
    listen 80;
    
    # server names (domains)
    server_name myapp.com www.myapp.com;
    
    # web root where your site files are located (static files)
    root /var/www/myapp.com/html;

    # pointing to the index file to serve by default, generally it always look for index.html in the given root dir specified by default if index file is different or so you can specify like below
    index index.html index.htm; 
    
    # location block to handle requests
    # try_files tries to serve the URI requested by the client first, here try_files $uri $uri/ =404; you're instructing NGINX to try for the requested URI first. If that doesn't work then try for the requested URI as a directory, and whenever NGINX ends up into a directory it automatically starts looking for an index.html file.
    location / {
        try_files $uri $uri/ =404;
    }

    # log files (optional), logs the access and error logs for this specific site
    access_log /var/log/nginx/myapp.access.log;
    error_log /var/log/nginx/myapp.error.log;
}
```
> There are many other ways we can configure the location via /=, regex etc based on our needs. Also we can add more location blocks for handling specific paths like /admin/, /static/ etc. depending on our application structure. Also we can add more directives inside the server block for further customization like setting up caching, headers, compression etc.

### Enable the site:

```bash
# Create symlink to enable
sudo ln -s /etc/nginx/sites-available/myapp.com.conf /etc/nginx/sites-enabled/myapp.com.conf

# Verify
ls -lh /etc/nginx/sites-enabled/

# Remove default site (optional)
sudo rm /etc/nginx/sites-enabled/default

# Test and reload
sudo nginx -t
sudo nginx -s reload
# OR
sudo systemctl reload nginx
```

---

## 6. Reverse Proxy Setup

This is the most common use case - proxying requests to our applications.

### Basic reverse proxy:

Below is a reverse proxy config to forward requests to a backend app running on localhost:8000 with websocket support, essential headers and timeouts configured. 

> When using Nginx as a reverse proxy, it's important to use `proxy_pass` directive to forward requests to desired service. Additionally, setting appropriate headers like `Host`, `X-Real-IP`, `X-Forwarded-For`, and `X-Forwarded-Proto` is crucial for preserving client information and ensuring proper handling by the backend service. WebSocket support is enabled by including the necessary headers to upgrade the connection. Timeouts are configured to handle long-running requests effectively.

```nginx
server {
    listen 80;
    server_name api.myapp.com;

    location / {
        proxy_pass http://127.0.0.1:8000;  # Your backend app
        proxy_http_version 1.1;
        
        client_max_body_size 100m; # Max upload size

        # Headers
        # This headers will be added to the current request before passing it to the backend server
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header REMOTE-HOST $remote_addr;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_redirect off;  # Disable automatic redirects i.e if backend sends a redirect response, Nginx will not modify the location header in the response
        proxy_buffering off;

        # Timeouts
        proxy_connect_timeout       600s;  # Time to establish connection with backend
        proxy_send_timeout          600s;  # Time to send request to backend
        proxy_read_timeout          600s;  # Time to read response from backend
        send_timeout                600s;  # Time to send response to client
        keepalive_timeout 600s;            # Keepalive timeout for upstream connections
    }
}
```

### Multiple Services:

> In below example, we have a frontend app (React/Vue/Angular) running on port 3000 and a backend API (Django/FastAPI) running on port 8000. Nginx will route requests based on the path. If user visit `www.myapp.com/`, they get the frontend app. If they visit `www.myapp.com/api/`, requests are proxied to the backend API. Generally backend apis will be in subdomain like `api.myapp.com` & frontend in `myapp.com` in that case we would create separate server blocks for each with location block as `/` only. We can also create seperate site configs for each and enable them separately via symlinks in `sites-enabled/`.

```nginx
server {
    listen 80;
    server_name myapp.com;

    # Frontend (React/Vue/Angular)
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Backend API (Django/FastAPI)
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 7. SSL/TLS with Let's Encrypt (X)

### Install Certbot:

```bash
# Install Certbot and Nginx plugin
sudo apt install certbot python3-certbot-nginx -y
```

### Obtain SSL certificate:

```bash
# Get certificate and auto-configure Nginx
sudo certbot --nginx -d myapp.com -d www.myapp.com
```

### Prompts:
```
Enter email address: your@email.com
Agree to terms: Y
Share email with EFF: N (optional)
Redirect HTTP to HTTPS: 2 (Yes, redirect)
```

### What Certbot does:
1. Verifies domain ownership
2. Obtains certificate from Let's Encrypt
3. Modifies your Nginx config to use SSL
4. Sets up auto-redirect from HTTP to HTTPS
5. Configures auto-renewal

### Verify auto-renewal:

```bash
# Test renewal process
sudo certbot renew --dry-run

# Check renewal timer
sudo systemctl status certbot.timer
```

---

## 8. Security Headers

### Create a reusable snippet:

> Generally we would need to add security headers to our server blocks for better security. Instead of adding them in each server block, we can create a reusable snippet and include it wherever needed using `include` directive as similar we included mime.types file in main nginx.conf

```bash
sudo nano /etc/nginx/snippets/security-headers.conf
```

```nginx
# Security Headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https:;" always;
add_header Permissions-Policy "accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=()" always;

# Remove server header (additional to server_tokens off)
more_clear_headers Server;
```

### Use in server block:

```nginx
server {
    listen 443 ssl;
    server_name myapp.com;
    
    include snippets/security-headers.conf;
    
    # ... rest of configuration
}
```

### Headers explained:

| Header | Purpose |
|--------|---------|
| X-Frame-Options | Prevent clickjacking |
| X-Content-Type-Options | Prevent MIME sniffing |
| X-XSS-Protection | XSS filter (legacy browsers) |
| Referrer-Policy | Control referrer information |
| Content-Security-Policy | Control resource loading |
| Permissions-Policy | Control browser features |

---

## 9. Performance Optimization

### Worker Processes & Connections

In ngnix config file, set worker processes and connections:

```nginx
worker_processes auto;  # Auto-detect CPU cores

events {
    worker_connections 1024;  # Max connections per worker
}
```

### Gzip compression (add to http block):

By writing `gzip on` in the http context, you're instructing NGINX to compress responses. By default, gzip configuration is already present in nginx.conf you can modify it as per your needs. Keep `gzip_comp_level` between 1-4 (uncomment if commented). By default, NGINX compresses HTML responses. To compress other file formats, you'll have to pass them as parameters to the `gzip_types` directive. You can just uncomment it in the config and modify as per your needs.

> Note: Enabling gzip is not enough to compress responses. The client (browser) must also support gzip compression and send the appropriate `Accept-Encoding` header in the request. NGINX checks for this header before compressing the response. For example, if a client request css file without `Accept-Encoding: gzip`, NGINX will serve the file uncompressed even if gzip is enabled. So to get the benefits of gzip compression, ensure that clients send the correct headers. Ref: `curl -I -H "Accept-Encoding: gzip" http://domain/mini.min.css`

### Static file caching:

Regardless of the application you're serving, there is always a certain amount of static content being served, such as stylesheets, images, and so on.

```nginx
# Add caching configs to location blocks serving static files (of backend or static site)
location /static/ {
    alias /var/www/myapp/static/;
    
    # Cache settings
    add_header Cache-Control public;
    add_header Vary Accept-Encoding;
    expires 1M;
    
    # Disable access logging for static files
    access_log off;
}

# Media files also same way
location /media/ {
    alias /var/www/myapp/media/;
    
    # Cache settings
    add_header Cache-Control public;
    add_header Vary Accept-Encoding;
    expires 1M;
    
    # Disable access logging for static files
    access_log off;
}

# Cache specific file types
location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|woff|woff2)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    add_header Vary Accept-Encoding;
    access_log off;
}
```

> By writing `location ~* .(css|js|jpg)$` you're instructing NGINX to match requests asking for a file ending with .css, .js and .jpg. The matched files will be served with the specified caching headers.

> You can use the `add_header` directive to include a header in the response to the client. We use `proxy_set_header` directive for setting headers on an ongoing request to the backend server. The `add_header` directive on the other hand only adds a given header to the response back to the client.

> By setting the `Cache-Control` header to public, you're telling the client that this content can be cached in any way.

> `Vary`, is responsible for letting the client know that this cached content may vary. Its Value `Accept-Encoding` means that the content may vary depending on the content encoding accepted by the client. (This header lets the client know that the response may vary based on what the client accepts.)

> The `expires` directive allows you to set the Expires header conveniently. The expires directive takes the duration of time this cache will be valid. By setting it to 1M you're telling NGINX to cache the content for one month. You can also set this to 10m or 10 minutes, 24h or 24 hours, and so on.

### Buffer optimization:

Buffer sizes can be adjusted based on your application needs. Below are some common buffer settings to optimize performance. Also adjust timeouts as per your requirements. We can add this to http block in nginx.conf or inside specific server blocks as needed as per the application.

```nginx
# Buffers Configuration
client_body_buffer_size 10K;
client_header_buffer_size 1k;
client_max_body_size 100m;  # Max upload size
large_client_header_buffers 4 32k;

# Timeouts Configuration
client_body_timeout 12;
client_header_timeout 12;
proxy_connect_timeout 600s;
proxy_send_timeout 600s;
proxy_read_timeout 600s;
send_timeout 600s;
keepalive_timeout 600s;
```

---

## 10. Load Balancing

In a real life scenario, load balancing may be required on large scale projects distributed across multiple servers

### Basic load balancing:

```nginx
upstream backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}

server {
    listen 80;
    server_name myapp.com;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> An `upstream` in NGINX is a collection of servers that can be treated as a single backend. So the three servers defined inside the upstream block can be treated as a single backend server named `backend`. When a request comes to NGINX, it will forward the request to one of the servers defined in the upstream block based on the load balancing method used (default is round-robin). Thus load is distributed across multiple servers by Nginx itself.

### Load balancing methods:

```nginx
# Round Robin (default)
upstream backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

# Least Connections
upstream backend {
    least_conn;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

# IP Hash (sticky sessions)
upstream backend {
    ip_hash;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

# Weighted
upstream backend {
    server 127.0.0.1:8001 weight=3;  # Gets 3x traffic
    server 127.0.0.1:8002 weight=1;
}

# With health checks
upstream backend {
    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8002 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8003 backup;  # Only used when others fail
}
```

---

## 11. Common Configurations

Below are some common Nginx configurations for different application types.

### Django Application (with SSL):

```nginx
# HTTP to HTTPS redirect for api.myapp.com (Automated by Certbot CLI when installing cert)
server {
    if ($host = api.myapp.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    listen [::]:80;
    server_name api.myapp.com;
    return 404; # managed by Certbot
}

server {
    server_name api.myapp.com;

    # SSL configuration (automated by Certbot)
    listen [::]:443 ssl; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/api.myapp.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/api.myapp.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    # Django app
    location / {
        proxy_pass http://127.0.0.1:8000;
        client_max_body_size 100m;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_redirect off;
        proxy_buffering off;
        proxy_connect_timeout       600s;
        proxy_send_timeout          600s;
        proxy_read_timeout          600s;
        send_timeout                600s;
        keepalive_timeout 600s;
    }

    # Static files
    location /static/ {
        proxy_pass http://127.0.0.1:8000/static/;
        expires 30d;
        access_log off;
    }

    # Media files
    location /media/ {
        proxy_pass http://127.0.0.1:8000/media/;
        expires 7d;
    }

    # Log files
    access_log  /var/log/nginx/api.myapp.com.log;
    error_log  /var/log/nginx/api.myapp.com.error.log;
}
```

### React/Vue SPA with API: (X)

```nginx
server {
    listen 443 ssl http2;
    server_name myapp.com;

    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    root /var/www/myapp/frontend/dist;
    index index.html;

    # SPA - serve index.html for all routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
}
```

---

## 12. Logging

Nginx logs requests and errors to log files for monitoring and debugging. You can customize logging as needed.

### Custom log format:

For detailed logging, define a custom log format in the http block:

```nginx
# In http block of nginx.conf
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time $upstream_response_time';

access_log /var/log/nginx/access.log detailed;
```

### Per-site logging:

For specific site logging, set log paths in server block:

```nginx
server {
    listen 80;
    server_name myapp.com;

    access_log /var/log/nginx/myapp.access.log;
    error_log /var/log/nginx/myapp.error.log;

    # ...
}
```

### Disable logging for specific paths:

You can disable logging for certain locations to reduce log size:

```nginx
location /health {
    access_log off;
    return 200 "OK";
}

location /static/ {
    access_log off;
    # ...
}
```

### View logs:

Nginx logs are located in `/var/log/nginx/` by default. If you set custom paths, check those locations. You can use `tail` and `grep` to view and filter logs:

```bash
# Access logs
sudo tail -f /var/log/nginx/access.log

# Error logs
sudo tail -f /var/log/nginx/error.log

# Specific site
sudo tail -f /var/log/nginx/myapp.error.log

# Search for errors
sudo grep "error" /var/log/nginx/error.log
```

---

## 📌 Quick Reference

```bash
# ========== SERVICE MANAGEMENT ==========
sudo systemctl start nginx     # Start
sudo systemctl stop nginx      # Stop
sudo systemctl restart nginx   # Restart (drops connections)
sudo systemctl reload nginx    # Reload (graceful, no downtime)
sudo systemctl status nginx    # Status

# ========== CONFIGURATION ==========
sudo nginx -t                  # Test configuration
sudo nginx -T                  # Test and dump configuration
sudo nginx -s reload           # Reload (alternative)
sudo nginx -s quit             # Graceful shutdown
sudo nginx -s stop             # Fast shutdown

# ========== LOGS ==========
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
sudo cat /var/log/nginx/error.log | grep "error"

# ========== SSL ==========
sudo certbot --nginx -d domain.com   # Get certificate
sudo certbot renew                   # Renew certificates
sudo certbot renew --dry-run         # Test renewal
sudo certbot certificates            # List certificates

# ========== SITES ==========
# Create
sudo nano /etc/nginx/sites-available/mysite

# Enable site
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/

# Disable site
sudo rm /etc/nginx/sites-enabled/mysite

# Test & reload
sudo nginx -t && sudo systemctl reload nginx

# ========== USEFUL ==========
sudo nginx -V                  # Version and compile options
sudo ss -tlnp | grep nginx     # Check listening ports
```

---
