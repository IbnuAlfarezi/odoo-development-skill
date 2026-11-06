# Docker & Docker Compose Deployment Patterns for Odoo

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  DOCKER & DOCKER COMPOSE PATTERNS FOR ODOO                                   ║
║  Side-by-side comparison of Production Stack vs Development Stack,           ║
║  Volume persistence, Network isolation, Healthchecks, and Remote Debugging  ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Containerizing Odoo provides environment reproducibility across machines. However, a **Production Docker Stack** and a **Development Docker Stack** have conflicting architectural goals:

- **Production Stack**: Emphasizes network isolation (database is hidden on private network), named volume persistence, automated restarts, healthchecks, and non-root execution.
- **Development Stack**: Emphasizes live bind-mounting of local source code, exposed database ports for GUI clients, and open `debugpy` ports for interactive IDE breakpoints.

---

## 1. Production Docker Compose (`docker-compose.prod.yml`)

This multi-container architecture isolates the database completely from the public internet. Only Nginx binds to ports `80` and `443`.

```yaml
version: "3.8"

services:
  # -------------------------------------------------------------------
  # 1. Database Tier (Isolated on internal network)
  # -------------------------------------------------------------------
  postgres:
    image: postgres:16-alpine
    container_name: odoo_prod_db
    restart: always
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - odoo_db_data:/var/lib/postgresql/data/pgdata
      - ./config/postgresql.conf:/etc/postgresql/postgresql.conf:ro
    command: ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
    networks:
      - internal_backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U odoo"]
      interval: 10s
      timeout: 5s
      retries: 5

  # -------------------------------------------------------------------
  # 2. Application Server Tier (Odoo Core)
  # -------------------------------------------------------------------
  odoo:
    image: odoo:18.0
    container_name: odoo_prod_app
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      - HOST=postgres
      - USER=odoo
      - PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - odoo_filestore_data:/var/lib/odoo
      - ./config/odoo.conf:/etc/odoo/odoo.conf:ro
      - ./custom_addons:/mnt/extra-addons:ro
    networks:
      - internal_backend
      - proxy_tier

  # -------------------------------------------------------------------
  # 3. Edge Reverse Proxy Tier (Nginx SSL & WebSocket)
  # -------------------------------------------------------------------
  nginx:
    image: nginx:alpine
    container_name: odoo_prod_proxy
    restart: always
    depends_on:
      - odoo
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /var/log/nginx:/var/log/nginx
    networks:
      - proxy_tier

networks:
  internal_backend:
    internal: true # No external internet routing to PostgreSQL
  proxy_tier:
    driver: bridge

volumes:
  odoo_db_data:
    driver: local
  odoo_filestore_data:
    driver: local
```

---

## 2. Development Docker Compose (`docker-compose.dev.yml`)

This configuration binds local folders directly, exposes PostgreSQL port `5432` to the host, and starts Odoo with `debugpy` on port `5678`:

```yaml
version: "3.8"

services:
  # Database exposed to localhost for TablePlus / DBeaver / pgAdmin
  postgres:
    image: postgres:16-alpine
    container_name: odoo_dev_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo
    ports:
      - "5432:5432"
    volumes:
      - odoo_dev_pgdata:/var/lib/postgresql/data

  # Odoo Application with live code editing and debugpy
  odoo:
    image: odoo:18.0
    container_name: odoo_dev_app
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      - HOST=postgres
      - USER=odoo
      - PASSWORD=odoo
    ports:
      - "8069:8069" # Main HTTP web client
      - "5678:5678" # debugpy remote debugger port
    volumes:
      # Live bind mount local addons (changes reflect immediately)
      - ./custom_addons:/mnt/extra-addons
      - ./config/odoo.dev.conf:/etc/odoo/odoo.conf
      - odoo_dev_filestore:/var/lib/odoo
    command: >
      python3 -m debugpy --listen 0.0.0.0:5678 --wait-for-client
      /usr/bin/odoo-bin
      -c /etc/odoo/odoo.conf
      --dev=reload,qweb,xml

volumes:
  odoo_dev_pgdata:
  odoo_dev_filestore:
```

---

## 3. Remote IDE Debugger Attachment (`.vscode/launch.json`)

To set breakpoints in your local editor (VS Code, Cursor, Windsurf, Antigravity) that execute inside the Docker container:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Odoo: Attach to Docker (Port 5678)",
      "type": "debugpy",
      "request": "attach",
      "connect": {
        "host": "localhost",
        "port": 5678
      },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}/custom_addons",
          "remoteRoot": "/mnt/extra-addons"
        }
      ],
      "justMyCode": false
    }
  ]
}
```

---

## 4. Production vs Development Docker Matrix

| Property | Development Stack | Production Stack |
| :--- | :--- | :--- |
| **Compose File** | `docker-compose.dev.yml` | `docker-compose.prod.yml` |
| **Addons Mount** | Live bind mount (`- ./custom_addons:/mnt/...`) | Read-only bind mount or baked into Docker image |
| **Port Exposure** | Direct `8069:8069` + `5432:5432` + `5678:5678` | **Only Nginx 80 & 443** exposed |
| **Network Type** | Bridge network | Private `internal: true` backend network |
| **Process Debugger**| `debugpy` enabled | **Disabled** (multiprocessing enabled) |
| **Restart Policy** | `unless-stopped` | `always` |
| **Data Volumes** | Ephemeral or local named volumes | High-speed dedicated persistent volumes |
