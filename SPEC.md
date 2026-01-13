# Docker Compose File Specification

This document defines the standards and conventions for all docker-compose.yaml files in this repository.

## Directory Structure

### Root Filesystem Layout

All services follow a standardized rootfs structure:

```
./rootfs/
├── config/          # Configuration files
├── data/            # Application data
│   ├── log/         # Log files (centralized like /var/log)
│   ├── media/       # Media files (movies, tv, music, etc.)
│   └── downloads/   # Download files
└── db/              # Database files
```

### Volume Path Conventions

#### Primary Services (app/web/server)
Primary services use the **project name** for their directories:

```yaml
# Project: gotify, Service: app (primary)
volumes:
  - './rootfs/data/gotify:/app/data'
  - './rootfs/config/gotify:/config'

# Project: authentik, Service: server (primary)
volumes:
  - './rootfs/data/authentik/media:/media'
  - './rootfs/data/authentik/templates:/templates'
  - './rootfs/config/authentik:/config'
```

#### Sub-Services
Sub-services use their **actual service name** (with project prefix stripped):

```yaml
# Project: authentik, Service: authentik-worker
volumes:
  - './rootfs/config/worker/certs:/certs'
  - './rootfs/data/authentik/templates:/templates'

# Project: jitsi, Service: jitsi-prosody
volumes:
  - './rootfs/config/prosody:/config'
  - './rootfs/data/prosody:/prosody-plugins-custom'
```

#### Database Services
Databases always use the pattern: `./rootfs/db/{dbtype}/{projectname}`

```yaml
# Project: authentik, Service: authentik-redis
volumes:
  - './rootfs/db/redis/authentik:/data'

# Project: authentik, Service: authentik-db (postgres)
volumes:
  - './rootfs/db/postgres/authentik:/var/lib/postgresql/data'

# Project: ampache, Service: ampache-db (mariadb)
volumes:
  - './rootfs/db/mariadb/ampache:/var/lib/mysql'
```

#### Shared Directories
When services share data, both mount the same primary path:

```yaml
# Server and worker both need templates
server:
  volumes:
    - './rootfs/data/authentik/templates:/templates'

worker:
  volumes:
    - './rootfs/data/authentik/templates:/templates'
```

#### Common Data Directories
For media, downloads, logs, or other shared content:

```yaml
# Media files - use lowercase for everything
volumes:
  - './rootfs/data/media/movies:/movies'
  - './rootfs/data/media/tv:/tv'
  - './rootfs/data/media/music:/music'
  
# Downloads
volumes:
  - './rootfs/data/downloads:/downloads'
  
# Logs - centralized like /var/log
volumes:
  - './rootfs/data/log/servicename:/var/log/servicename'
  - './rootfs/data/log/nginx:/var/log/nginx'
```

### Volume Mount Simplification

**Always use what the container expects for internal mount paths.**

Do NOT simplify mounts when containers expect different internal paths:

```yaml
# CORRECT - Keep separate when internal paths differ
volumes:
  - './rootfs/data/authentik/media:/media'
  - './rootfs/data/authentik/templates:/templates'

# WRONG - Don't combine if container needs specific paths
volumes:
  - './rootfs/data/authentik:/app'
```

Simplify only when the container expects a single base path:

```yaml
# CORRECT - Single mount when container uses one base
volumes:
  - './rootfs/data/gotify:/app/data'
```

## Container Naming

Container names always follow the pattern: `{projectname}-{servicename}`

```yaml
# Project: authentik
services:
  server:
    container_name: authentik-server
  
  worker:
    container_name: authentik-worker
  
  redis:
    container_name: authentik-redis
  
  db:
    container_name: authentik-db

# Project: jitsi
services:
  web:
    container_name: jitsi-web
  
  prosody:
    container_name: jitsi-prosody
  
  jicofo:
    container_name: jitsi-jicofo
```

## Network Configuration

### Network Structure
Each project defines only its own internal network as `external: false`.

**Remove all external proxy and cloudflare network declarations.**

```yaml
# CORRECT
networks:
  gotify:
    name: gotify
    external: false

# WRONG - Do not include external networks
networks:
  gotify:
    name: gotify
    external: false
  proxy:
    external: true
  cloudflare:
    external: true
```

### Service Network Membership
Services should only be members of their project network:

```yaml
services:
  app:
    networks:
      - gotify

networks:
  gotify:
    name: gotify
    external: false
```

## Environment Variables

All environment variables must have **sane defaults** to work without .env files. This allows running `docker compose up` immediately without configuration.

```yaml
environment:
  - TZ=${TZ:-America/New_York}
  - HOSTNAME=${BASE_HOST_NAME:-$HOSTNAME}
  - DB_PASSWORD=${DB_ADMIN_PASS:-changeme123}
```

### Common Variables

#### Base Configuration
- `TZ` - Timezone (default: `America/New_York`)
- `BASE_DOMAIN_NAME` - Primary domain name (default: varies by context)
- `BASE_HOST_NAME` - Hostname (default: `$HOSTNAME`)

#### Database Configuration
- `DB_ADMIN_NAME` - Database admin username (default: `admin` or `postgres`)
- `DB_ADMIN_PASS` - Database admin password (default: generate random or use placeholder)
- `DB_USER_NAME` - Database user username (default: varies)
- `DB_USER_PASS` - Database user password (default: varies)
- `DB_CREATE_DATABASE_NAME` - Database name to create (default: projectname)

#### Application Security
- `APP_ADMIN_USER` - Application admin username (default: `admin`)
- `APP_ADMIN_PASS` - Application admin password (default: generate random)
- `APP_USER_NAME` - Application user username (default: varies)
- `APP_USER_PASS` - Application user password (default: varies)
- `APP_SECRET_KEY` - Application secret key (default: generate random)
- `APP_SECRET_TOKEN_16` - 16-character secret token (default: generate random)
- `APP_SECRET_TOKEN_32` - 32-character secret token (default: generate random)
- `APP_SECRET_TOKEN_64` - 64-character secret token (default: generate random)
- `APP_API_TOKEN` - API token (default: generate random)
- `APP_JWT_TOKEN` - JWT token (default: generate random)
- `ENCRYPTION_KEY` - Encryption key (default: generate random)
- `RPC_SECRET` - RPC secret (default: generate random)

#### Email/SMTP Configuration
**Use these standardized SMTP settings for all projects:**

- `EMAIL_SERVER_HOST` - SMTP host (default: `172.17.0.1` - docker gateway)
- `EMAIL_SERVER_PORT` - SMTP port (default: `587`)
- `EMAIL_SERVER_TIMEOUT` - SMTP timeout in seconds (default: `10`)
- `EMAIL_SERVER_FROM_ORG` - From organization name (default: `{PROJECTNAME}`, e.g., "Gotify", "Authentik")
- `EMAIL_SERVER_MAIL_FROM` - From email address (default: `no-reply@${BASE_DOMAIN_NAME:-${BASE_HOST_NAME}}`)
- `EMAIL_SERVER_LOGIN_NAME` - SMTP username (optional, default: empty)
- `EMAIL_SERVER_LOGIN_PASS` - SMTP password (optional, default: empty)

**Example SMTP configuration:**

```yaml
environment:
  # Email settings
  - EMAIL_HOST=${EMAIL_SERVER_HOST:-172.17.0.1}
  - EMAIL_PORT=${EMAIL_SERVER_PORT:-587}
  - EMAIL_FROM_NAME=${EMAIL_SERVER_FROM_ORG:-Authentik}
  - EMAIL_FROM=${EMAIL_SERVER_MAIL_FROM:-no-reply@${BASE_DOMAIN_NAME:-${BASE_HOST_NAME}}}
  - EMAIL_USER=${EMAIL_SERVER_LOGIN_NAME:-}
  - EMAIL_PASS=${EMAIL_SERVER_LOGIN_PASS:-}
  - EMAIL_TIMEOUT=${EMAIL_SERVER_TIMEOUT:-10}
```

#### Organization
- `APP_ORG_NAME` - Organization name (default: varies by project)

#### Cloudflare (for dockflare/proxy services)
- `CLOUDFLARE_EMAIL` - Cloudflare account email
- `CLOUDFLARE_API_KEY` - Cloudflare API key
- `CLOUDFLARE_ZONE_ID` - Cloudflare zone ID
- `CLOUDFLARE_ZONE_NAME` - Cloudflare zone name
- `CLOUDFLARE_TUNNEL_ID` - Cloudflare tunnel ID
- `CLOUDFLARE_ACCOUNT_ID` - Cloudflare account ID
- `TRUSTED_PROXIES` - Trusted proxy IPs (default: `127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16`)

### Generating Secure Defaults

For passwords, secrets, and tokens, use these patterns in defaults:

```bash
# Random secret (16 characters)
openssl rand -hex 8

# Random password (32 characters)
head -n50 '/dev/random' | tr -dc 'a-zA-Z0-9[!@-]' | tr -d '[:space:]\042\047\134' | fold -w 32 | head -n 1

# Hashed password
head -n128 /dev/urandom|tr -dc 'A-Za-z0-9' | head -c 32|openssl passwd -1 -salt $(openssl rand -base64 6) --stdin
```

**In compose files, use placeholder defaults that are secure enough for development but obviously need changing for production:**

```yaml
environment:
  - DB_ADMIN_PASS=${DB_ADMIN_PASS:-changeme_db_admin_password}
  - APP_SECRET_KEY=${APP_SECRET_KEY:-changeme_secret_key_min_32_chars}
  - APP_API_TOKEN=${APP_API_TOKEN:-changeme_api_token}
```

## Code Style

### Comments
Comments always go **above** the line, never inline:

```yaml
# CORRECT
# This is the main application service
services:
  app:
    image: myapp:latest

# WRONG
services:
  app:
    image: myapp:latest  # This is the main application service
```

### Case Convention
Use lowercase for all directory names and paths unless specifically required:

```yaml
# CORRECT
- './rootfs/data/media/movies:/movies'
- './rootfs/config/nginx:/etc/nginx'

# WRONG
- './rootfs/data/Media/Movies:/movies'
- './rootfs/Data/NGINX:/etc/nginx'
```

### Standard File Structure
All compose files should follow this order:

1. Comment header (nginx proxy address, notes, etc.)
2. `name:` declaration
3. `x-logging:` anchor (if used)
4. `x-env:` anchor (if used)
5. `services:` section
6. `networks:` section

```yaml
# nginx proxy address - http://172.17.0.1:62084

name: gotify
x-logging: &default-logging
  options:
    max-size: '5m'
    max-file: '1'
  driver: json-file

services:
  app:
    image: gotify/server:latest
    container_name: gotify-app
    # ... service configuration

networks:
  gotify:
    name: gotify
    external: false
```

## Service Configuration Standards

### Standard Service Properties
Every service should include (where applicable):

```yaml
services:
  app:
    image: service:latest
    container_name: projectname-app
    hostname: ${BASE_HOST_NAME:-$HOSTNAME}
    restart: always
    pull_policy: always
    logging: *default-logging
    environment:
      - TZ=${TZ:-America/New_York}
      - CONTAINER_NAME: projectname-app
      - HOSTNAME: ${BASE_HOST_NAME:-$HOSTNAME}
    ports:
      - '172.17.0.1:port:containerport'
    volumes:
      - './rootfs/data/projectname:/data'
    networks:
      - projectname
```

## Examples

### Simple Single Service (gotify)

```yaml
# nginx proxy address - http://172.17.0.1:62084

name: gotify
x-logging: &default-logging
  options:
    max-size: '5m'
    max-file: '1'
  driver: json-file

services:
  app:
    image: gotify/server:latest
    container_name: gotify-app
    hostname: ${BASE_HOST_NAME:-$HOSTNAME}
    restart: always
    pull_policy: always
    logging: *default-logging
    environment:
      TZ: ${TZ:-America/New_York}
      CONTAINER_NAME: gotify-app
      HOSTNAME: ${BASE_HOST_NAME:-$HOSTNAME}
    ports:
      - '172.17.0.1:62084:80'
    volumes:
      - './rootfs/data/gotify:/app/data:z'
    networks:
      - gotify

networks:
  gotify:
    name: gotify
    external: false
```

### Complex Multi-Service (authentik)

```yaml
# nginx proxy address - http://172.17.0.1:62000

name: authentik
x-logging: &default-logging
  options:
    max-size: '5m'
    max-file: '1'
  driver: json-file

services:
  server:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-server
    hostname: ${BASE_HOST_NAME:-$HOSTNAME}
    restart: always
    pull_policy: always
    logging: *default-logging
    volumes:
      - './rootfs/data/authentik/media:/media'
      - './rootfs/data/authentik/templates:/templates'
    networks:
      - authentik

  worker:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-worker
    hostname: authentik-worker
    restart: always
    logging: *default-logging
    volumes:
      - './rootfs/config/worker/certs:/certs'
      - './rootfs/data/authentik/media:/media'
      - './rootfs/data/authentik/templates:/templates'
    networks:
      - authentik

  redis:
    image: redis:alpine
    container_name: authentik-redis
    hostname: authentik-redis
    restart: always
    logging: *default-logging
    volumes:
      - './rootfs/db/redis/authentik:/data'
    networks:
      - authentik

  db:
    image: postgres:16-alpine
    container_name: authentik-db
    hostname: authentik-db
    restart: always
    logging: *default-logging
    volumes:
      - './rootfs/db/postgres/authentik:/var/lib/postgresql/data'
    networks:
      - authentik

networks:
  authentik:
    name: authentik
    external: false
```

### Multi-Component Service (jitsi)

```yaml
# nginx proxy address - http://172.17.0.1:60236

name: jitsi
x-logging: &default-logging
  options:
    max-size: '5m'
    max-file: '1'
  driver: json-file

services:
  web:
    image: jitsi/web:latest
    container_name: jitsi-web
    restart: always
    logging: *default-logging
    volumes:
      - './rootfs/config/jitsi:/config:Z'
      - './rootfs/data/jitsi/transcripts:/usr/share/jitsi-meet/transcripts:Z'
    networks:
      - jitsi

  prosody:
    image: jitsi/prosody:latest
    container_name: jitsi-prosody
    restart: always
    logging: *default-logging
    volumes:
      - './rootfs/config/prosody:/config:Z'
      - './rootfs/data/prosody:/prosody-plugins-custom:Z'
    networks:
      - jitsi

  jicofo:
    image: jitsi/jicofo:latest
    container_name: jitsi-jicofo
    restart: always
    logging: *default-logging
    volumes:
      - './rootfs/config/jicofo:/config:Z'
    networks:
      - jitsi

  jvb:
    image: jitsi/jvb:latest
    container_name: jitsi-jvb
    restart: always
    logging: *default-logging
    volumes:
      - './rootfs/config/jvb:/config:Z'
    networks:
      - jitsi

networks:
  jitsi:
    name: jitsi
    external: false
```

## Quick Reference

### Identifying Service Type

**Primary Service** (use projectname in paths):
- Service name is: `app`, `web`, `server`, or similar generic name
- It's the main/frontend service of the project

**Sub-Service** (use servicename in paths):
- Service name is: `worker`, `prosody`, `jicofo`, `jvb`, etc.
- It's a supporting service with a specific role

**Database Service** (use db/{dbtype}/{projectname}):
- Service provides: postgres, mysql, mariadb, redis, mongodb, etc.
- Always follows database path convention

### Path Templates

```
Primary:     ./rootfs/{type}/{projectname}
Sub-Service: ./rootfs/{type}/{servicename}
Database:    ./rootfs/db/{dbtype}/{projectname}
Media:       ./rootfs/data/media/{mediatype}
Downloads:   ./rootfs/data/downloads
Logs:        ./rootfs/data/log/{servicename}
```

### Container Name Template

```
{projectname}-{servicename}
```

### Network Template

```yaml
networks:
  {projectname}:
    name: {projectname}
    external: false
```
