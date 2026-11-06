# Nginx Reverse Proxy & SSL Configuration Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  NGINX REVERSE PROXY & SSL PATTERNS FOR ODOO                                 ║
║  Dual-upstream routing (HTTP 8069 vs WebSocket 8072), SSL/TLS Termination,  ║
║  Session cookie security, Large payload buffers, and Static asset caching   ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Odoo requires an edge reverse proxy (such as Nginx) in front of its application workers. In Odoo 16+, real-time notifications (LiveChat, bus notifications, and POS updates) operate over native WebSockets. Nginx must route `/websocket` requests to Odoo's longpolling/gevent worker port (`8072`), while standard HTTP traffic is sent to port `8069`.

---

## 1. Complete Production Nginx Site Configuration

Save this file as `/etc/nginx/sites-available/odoo.conf` and create a symlink to `/etc/nginx/sites-enabled/`:

```nginx
# =====================================================================
# Upstream Definitions
# =====================================================================
# Main Odoo HTTP application server
upstream odoo_backend {
    server 127.0.0.1:8069;
}

# LiveChat & WebSocket server (Gevent port)
upstream odoo_websocket {
    server 127.0.0.1:8072;
}

# WebSocket upgrade map
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

# Content-Security-Policy for static image responses (Official Odoo Recommendation)
map $sent_http_content_type $content_type_csp {
    default "";
    ~image/ "default-src 'none'";
}

# Rate Limiting Zones (Anti-Brute Force & Anti-DDoS)
limit_req_zone $binary_remote_addr zone=odoo_login_zone:10m rate=5r/m;
limit_req_zone $binary_remote_addr zone=odoo_api_zone:10m rate=30r/m;

# =====================================================================
# Server Block: HTTP -> HTTPS Redirect
# =====================================================================
server {
    listen 80;
    listen [::]:80;
    server_name erp.mycompany.com;

    # Certbot ACME Challenge for automated SSL certificate renewal
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Permanent redirect to secure HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# =====================================================================
# Server Block: HTTPS Main Application
# =====================================================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name erp.mycompany.com;

    # SSL / TLS Parameters (Certbot / Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/erp.mycompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/erp.mycompany.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # =================================================================
    # Security Headers
    # =================================================================
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Frame-Options "SAMEORIGIN" always;

    # =================================================================
    # Timeouts & Buffer Tuning for Large File Uploads & BI Reports
    # =================================================================
    client_max_body_size 120M;          # Required for large backup restores or file attachments
    proxy_read_timeout 720s;            # Prevents 504 Gateway Timeouts during heavy report generation
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;
    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    # Logging
    access_log /var/log/nginx/odoo.access.log;
    error_log /var/log/nginx/odoo.error.log;

    # Gzip Compression
    gzip on;
    gzip_min_length 1000;
    gzip_types text/css text/scss text/plain text/xml application/xml application/json application/javascript;

    # =================================================================
    # Location 1: WebSocket Traffic (Chat, Live Notifications, POS)
    # =================================================================
    location /websocket {
        proxy_pass http://odoo_websocket;
        
        # Upgrade HTTP connection to WebSocket
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Client Identity Headers for Odoo's proxy_mode
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;

        # Enforce secure cookie flags (Requires Nginx >= 1.19.8)
        proxy_cookie_flags session_id samesite=lax secure;
    }

    # =================================================================
    # Location 2: Anti-Brute Force Protection on Login
    # =================================================================
    location /web/login {
        limit_req zone=odoo_login_zone burst=10 nodelay;
        proxy_pass http://odoo_backend;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_cookie_flags session_id samesite=lax secure;
    }

    # =================================================================
    # Location 3: Rate Limiting on External XML-RPC / JSON-RPC APIs
    # =================================================================
    location ~ ^/(xmlrpc|jsonrpc) {
        limit_req zone=odoo_api_zone burst=20 nodelay;
        proxy_pass http://odoo_backend;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # =================================================================
    # Location 4: Edge Restriction of Database Manager (Official Odoo Hardening)
    # =================================================================
    # Block public access to dangerous database creation/drop/restore screens
    location ~ ^/web/database/(manager|create|duplicate|drop|backup) {
        allow 10.0.0.0/8;          # Internal VPN
        allow 192.168.1.0/24;      # Office LAN
        deny all;                  # Block all public internet traffic
        
        proxy_pass http://odoo_backend;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # =================================================================
    # Location 5: Attachment Offloading via Nginx X-Accel (Official Odoo Deploy)
    # =================================================================
    # Delegates file serving directly to Nginx, saving Python worker RAM
    location /web/filestore {
        internal;
        alias /var/lib/odoo/filestore;
        add_header Content-Security-Policy $upstream_http_content_security_policy;
        add_header X-Content-Type-Options nosniff;
    }

    # =================================================================
    # Location 6: Standard Odoo Web Requests
    # =================================================================
    location / {
        proxy_pass http://odoo_backend;
        proxy_redirect off;

        # Pass true client identity to Odoo
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;

        proxy_cookie_flags session_id samesite=lax secure;
    }

    # =================================================================
    # Location 7: Static Asset Caching (JS, CSS, Images, Fonts)
    # =================================================================
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|woff|woff2|ttf|svg)$ {
        proxy_pass http://odoo_backend;
        proxy_cache_valid 200 60m;
        proxy_buffering on;
        expires 7d;
        add_header Cache-Control "public, no-transform";
        add_header Content-Security-Policy $content_type_csp;
    }
}
```

---

## 2. Setting Up SSL via Certbot (Let's Encrypt)

```bash
# 1. Install Certbot and Nginx plugin
sudo apt update && sudo apt install -y certbot python3-certbot-nginx

# 2. Request SSL Certificate
sudo certbot --nginx -d erp.mycompany.com

# 3. Test Automated Renewal
sudo certbot renew --dry-run
```

---

## 3. Fail2ban Integration for Automated Login IP Banning

Configure Fail2ban to parse `/var/log/odoo/odoo-server.log` and automatically drop IP addresses that repeatedly fail authentication:

### Filter: `/etc/fail2ban/filter.d/odoo-login.conf`
```ini
[Definition]
failregex = ^ \d+ INFO \S+ \S+ Login failed for db:\S+ user:\S+ from <HOST>
ignoreregex =
```

### Jail: `/etc/fail2ban/jail.d/odoo.conf`
```ini
[odoo-login]
enabled = true
port = http,https
filter = odoo-login
logpath = /var/log/odoo/odoo-server.log
maxretry = 5
findtime = 600
bantime = 3600
action = iptables-multiport[name=Odoo, port="http,https"]
```
sudo certbot renew --dry-run
```

---

## 3. Critical Verification Steps

1. **Verify WebSocket Upgrade**:
   Open browser DevTools → Network tab → filter by `WS`. Reload page. You must see `/websocket` returning status code `101 Switching Protocols`. If it returns `400` or `502`, check that port `8072` is listening.
2. **Verify Client IP Resolution**:
   In Python backend, check `request.httprequest.remote_addr`. If it returns `127.0.0.1`, verify that `proxy_mode = True` is enabled in `odoo.conf` and `proxy_set_header X-Forwarded-For` is present in Nginx.
3. **Verify Upload Limits**:
   If uploading attachments larger than 1MB fails with `413 Request Entity Too Large`, confirm that `client_max_body_size` is explicitly configured in Nginx.

