# Docker Compose File Specification

This document defines the standards and conventions for all Docker Compose repositories in the organization.

## Source of truth

This document is authoritative for every repository in the organization. Keep
each repository's README, environment samples, compose files, and image
references consistent with the actual upstream application. Reference stacks
live in `.github/example/`.

## Repository layout

Compose files stay flat at the child repository root. Do not move them under a
`docker/` directory. Runtime files and state are untracked:

- `.env`
- `app.env`
- `default.env`
- `volumes/`

Each child repository has:

- `docker-compose.yaml`
- `docker-compose.traefik.yaml`
- `docker-compose.tunnel.yaml`
- `app.env.sample`
- `default.env.sample`
- `.gitignore`
- `README.md`

## Compose variants

The three compose files are complete, standalone alternatives. They must not
be combined with multiple `-f` arguments.

- `docker-compose.yaml`: default host-published mode; first line is
  `# nginx proxy address - http://172.17.0.1:{port}`; no proxy labels or
  external proxy network.
  A project with no HTTP interface (database, broker, CLI tool) replaces that
  first line with a comment saying so and publishes only its native port.
- `docker-compose.traefik.yaml`: no HTTP `ports:`; app joins the project network
  and external `traefik` network; use Traefik labels.
- `docker-compose.tunnel.yaml`: no HTTP `ports:`; app joins the project network
  and external `cloudflare` network; use Cloudflare tunnel labels.

Every file defines the project network with `external: false`. Database and
cache services join only the project network in all variants. The app joins
the project network and, in proxy variants, the matching external network.

Use the shared `x-logging` JSON-file anchor, `pull_policy: always`, map-style
environment variables, and a quoted published port. The container-side port is
the app's own native port and varies per project; only the host-side port is
normalized into the `64xxx` range. Confirm each native port against the
upstream documentation rather than assuming.

## Host gateway and port binding rules

`172.17.0.1:` is the Docker host gateway bind address for production-shaped
HTTP/HTTPS exposure. Use it only for the default host-published
`docker-compose.yaml` variant, where nginx on the host proxies to the unique
external port. The address must not appear in the Traefik or Cloudflare
variants as a published port. It may still appear as a non-port default value,
such as an SMTP relay address. It must also not be used for database, cache,
debug, or internal-only service ports.

The host-side port is always unique per project and falls in `64000`-`64999`.
The container-side port is the app's native port and must be verified against
upstream documentation. Do not publish internal service ports to the host.

`docker-compose.traefik.yaml` and `docker-compose.tunnel.yaml` publish no HTTP
or HTTPS ports and never use a `172.17.0.1:` bind. Non-HTTP protocol ports the
image declares and the product needs (for example XMPP, Git SSH, DNS, Tor,
Minecraft, BitTorrent, SMTP/IMAP) cannot be proxied over HTTP, so they are
published identically in all three variants as plain `host:container` ports
(add `/udp` where needed). A port range may use `a-b:a-b`. Git SSH on `22` is
mapped to a unique non-privileged host port (`2222`, `2223`, ...) to avoid the
host's own sshd. Admin, API and cluster-internal ports (for example Caddy's
`2019`) are not published. Confirm the needed ports against the image's
`ExposedPorts` and upstream documentation.

## Database and cache services

For ordinary application stacks, database and cache services are internal-only.
In every compose variant they must join only the project's internal network.
They must never join `traefik` or `cloudflare`, must never have a `ports:`
section, and must never use a `172.17.0.1:` bind. The app connects to them over
Docker service names on the project network.

A database, cache, or similar infrastructure project is the exception: its
primary service is the product being deployed and may expose its native
protocol port directly, using the image's standard internal and external port
(for example `3306:3306` for MySQL/MariaDB or `5432:5432` for PostgreSQL).
Do not use the `172.17.0.1:` HTTP bind for that native protocol port. If the
same project includes a web management service such as phpMyAdmin, Adminer,
or a database web console, that web service uses the normal unique
`172.17.0.1:{port}:80` assignment and may use the proxy variants. Keep the
database service itself on the project network only; expose only the documented
native port and any separately assigned web-management port.

Use the required persistent volume layout for database services:

```yaml
volumes:
  - './volumes/db/{database-type}/{project_name}:/var/lib/{database-type}/data:z'
```

## Unique host ports

Every child project must have a unique external host port in the `64000`-
`64999` range. The port must be used only in that project's default compose
file and README access URL. Never reuse a port already assigned to another
project. Proxy variants do not publish ports.

Current allocation is recorded in `PORTS.md`. `.github/example/` is `64000`;
the repositories take `64100` onward in alphabetical order. Existing
assignments must be preserved unless deliberately reallocated across the
fleet.

## Images

Every image reference must be pullable from its stated registry and tag. An
invalid image is a bug. Use the `:latest` tag whenever the upstream image
provides it; users may pin an exact version in their local copy or environment
where supported. Do not invent image names or tags. Verify new and changed
references with `docker manifest inspect` before declaring work complete.

Use a real, pullable image rather than a fictional placeholder.

## Environment files

`default.env.sample` mirrors the variables used by composemgr and uses an
`export` prefix. `app.env.sample` is the app-specific override file and uses
plain `KEY=VALUE` syntax without `export`.

Document this raw Docker Compose workflow in every README:

```bash
cp default.env .env
docker compose --env-file .env up -d
```

Compose stacks must boot using inline defaults without an env file. Composemgr
handles `app.env` and `default.env` automatically; raw Docker Compose users
must pass `--env-file .env`.

Sample secrets and credentials must be obviously fake, such as
`replace-with-a-random-password`; never use realistic tokens or credentials.

## README requirements

Each README documents the project description, services and actual image,
installation options, environment files, the assigned `172.17.0.1:{port}` URL,
volumes, security guidance where relevant, logging, management commands,
requirements, and author information. Raw Docker Compose commands include
`--env-file .env` whenever they rely on `.env`.

## Verification

Before completing changes:

1. Parse every changed compose file with `docker compose config -q`.
2. Verify env-file resolution with `docker compose --env-file .env config` or
   an equivalent sample env file.
3. Verify every changed image reference with `docker manifest inspect`, and
   confirm the native port with `docker image inspect --format
   '{{json .Config.ExposedPorts}}'`.
4. Confirm the project's external port is unique across all child compose
   files and is in the `64000`-`64999` range.
5. Confirm no real secrets or runtime files are tracked.
6. Confirm the proxy variants publish no HTTP/HTTPS ports and no `172.17.0.1:` bind, and that every needed non-HTTP protocol port is published in all three variants.

`docker compose config -q` only proves the file parses. It does not prove the
stack boots, so report per repository what was verified and what was not.


## Directory Structure

### Root Filesystem Layout

All services follow a standardized rootfs structure:

```
./volumes/
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
  - './volumes/data/gotify:/app/data'
  - './volumes/config/gotify:/config'

# Project: authentik, Service: server (primary)
volumes:
  - './volumes/data/authentik/media:/media'
  - './volumes/data/authentik/templates:/templates'
  - './volumes/config/authentik:/config'
```

#### Sub-Services
Sub-services use their **actual service name** (with project prefix stripped):

```yaml
# Project: authentik, Service: authentik-worker
volumes:
  - './volumes/config/worker/certs:/certs'
  - './volumes/data/authentik/templates:/templates'

# Project: jitsi, Service: jitsi-prosody
volumes:
  - './volumes/config/prosody:/config'
  - './volumes/data/prosody:/prosody-plugins-custom'
```

#### Database Services
Databases always use the pattern: `./volumes/db/{dbtype}/{projectname}`

```yaml
# Project: authentik, Service: authentik-redis
volumes:
  - './volumes/db/redis/authentik:/data'

# Project: authentik, Service: authentik-db (postgres)
volumes:
  - './volumes/db/postgres/authentik:/var/lib/postgresql/data'

# Project: ampache, Service: ampache-db (mariadb)
volumes:
  - './volumes/db/mariadb/ampache:/var/lib/mysql'
```

#### Shared Directories
When services share data, both mount the same primary path:

```yaml
# Server and worker both need templates
server:
  volumes:
    - './volumes/data/authentik/templates:/templates'

worker:
  volumes:
    - './volumes/data/authentik/templates:/templates'
```

#### Common Data Directories
For media, downloads, logs, or other shared content:

```yaml
# Media files - use lowercase for everything
volumes:
  - './volumes/data/media/movies:/movies'
  - './volumes/data/media/tv:/tv'
  - './volumes/data/media/music:/music'
  
# Downloads
volumes:
  - './volumes/data/downloads:/downloads'
  
# Logs - centralized like /var/log
volumes:
  - './volumes/data/log/servicename:/var/log/servicename'
  - './volumes/data/log/nginx:/var/log/nginx'
```

### Volume Mount Simplification

**Always use what the container expects for internal mount paths.**

Do NOT simplify mounts when containers expect different internal paths:

```yaml
# CORRECT - Keep separate when internal paths differ
volumes:
  - './volumes/data/authentik/media:/media'
  - './volumes/data/authentik/templates:/templates'

# WRONG - Don't combine if container needs specific paths
volumes:
  - './volumes/data/authentik:/app'
```

Simplify only when the container expects a single base path:

```yaml
# CORRECT - Single mount when container uses one base
volumes:
  - './volumes/data/gotify:/app/data'
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

Every project ships **three standalone compose files**, never one file with conditional network membership:

| File | Purpose | Networks |
|------|---------|----------|
| `docker-compose.yaml` | Default, direct `172.17.0.1:{port}` publish | project only |
| `docker-compose.traefik.yaml` | Traefik reverse proxy | project + external `traefik` |
| `docker-compose.tunnel.yaml` | Cloudflare tunnel | project + external `cloudflare` |

Each project defines its own internal network as `external: false` in **all three** files.

```yaml
# CORRECT - docker-compose.yaml
networks:
  gotify:
    name: gotify
    external: false

# CORRECT - docker-compose.traefik.yaml
networks:
  gotify:
    name: gotify
    external: false
  traefik:
    external: true

# WRONG - never mix the proxy and tunnel networks into one file
networks:
  gotify:
    name: gotify
    external: false
  traefik:
    external: true
  cloudflare:
    external: true
```

### Service Network Membership

The **app** service joins the project network plus the external network matching the file it is in. **Database and cache services join only the project network in all three files** — they are never published through a proxy.

```yaml
# docker-compose.yaml
services:
  app:
    networks:
      - gotify
  db:
    networks:
      - gotify

# docker-compose.traefik.yaml
services:
  app:
    networks:
      - gotify
      - traefik
  db:
    networks:
      - gotify
```

## Environment Variables

All environment variables must have **sane defaults** to work without .env files. This allows running `docker compose up` immediately without configuration.

```yaml
environment:
  TZ: ${TZ:-America/New_York}
  HOSTNAME: ${BASE_HOST_NAME:-$HOSTNAME}
  DB_PASSWORD: ${DB_ADMIN_PASS:-changeme123}
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
  EMAIL_HOST: ${EMAIL_SERVER_HOST:-172.17.0.1}
  EMAIL_PORT: ${EMAIL_SERVER_PORT:-587}
  EMAIL_FROM_NAME: ${EMAIL_SERVER_FROM_ORG:-Authentik}
  EMAIL_FROM: ${EMAIL_SERVER_MAIL_FROM:-no-reply@${BASE_DOMAIN_NAME:-${BASE_HOST_NAME}}}
  EMAIL_USER: ${EMAIL_SERVER_LOGIN_NAME:-}
  EMAIL_PASS: ${EMAIL_SERVER_LOGIN_PASS:-}
  EMAIL_TIMEOUT: ${EMAIL_SERVER_TIMEOUT:-10}
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
  DB_ADMIN_PASS: ${DB_ADMIN_PASS:-changeme_db_admin_password}
  APP_SECRET_KEY: ${APP_SECRET_KEY:-changeme_secret_key_min_32_chars}
  APP_API_TOKEN: ${APP_API_TOKEN:-changeme_api_token}
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
- './volumes/data/media/movies:/movies'
- './volumes/config/nginx:/etc/nginx'

# WRONG
- './volumes/data/Media/Movies:/movies'
- './volumes/Data/NGINX:/etc/nginx'
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
# nginx proxy address - http://172.17.0.1:64189

name: gotify
x-logging: &default-logging
  options:
    max-size: '5m'
    max-file: '1'
  driver: json-file

services:
  app:
    image: gotify/server:latest
    container_name: gotify-server
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
      TZ: ${TZ:-America/New_York}
      CONTAINER_NAME: projectname-app
      HOSTNAME: ${BASE_HOST_NAME:-$HOSTNAME}
    ports:
      - '172.17.0.1:port:containerport'
    volumes:
      - './volumes/data/projectname:/data'
    networks:
      - projectname
```

## Examples

### Simple Single Service (gotify)

```yaml
# nginx proxy address - http://172.17.0.1:64189

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
      - '172.17.0.1:64189:80'
    volumes:
      - './volumes/data/gotify:/app/data:z'
    networks:
      - gotify

networks:
  gotify:
    name: gotify
    external: false
```

### Complex Multi-Service (authentik)

```yaml
# nginx proxy address - http://172.17.0.1:64115

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
      - './volumes/data/authentik/media:/media'
      - './volumes/data/authentik/templates:/templates'
    networks:
      - authentik

  worker:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-worker
    hostname: authentik-worker
    restart: always
    logging: *default-logging
    volumes:
      - './volumes/config/worker/certs:/certs'
      - './volumes/data/authentik/media:/media'
      - './volumes/data/authentik/templates:/templates'
    networks:
      - authentik

  redis:
    image: redis:alpine
    container_name: authentik-redis
    hostname: authentik-redis
    restart: always
    logging: *default-logging
    volumes:
      - './volumes/db/redis/authentik:/data'
    networks:
      - authentik

  db:
    image: postgres:16-alpine
    container_name: authentik-postgres
    hostname: authentik-db
    restart: always
    logging: *default-logging
    volumes:
      - './volumes/db/postgres/authentik:/var/lib/postgresql/data'
    networks:
      - authentik

networks:
  authentik:
    name: authentik
    external: false
```

### Multi-Component Service (jitsi)

```yaml
# nginx proxy address - http://172.17.0.1:64217

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
      - './volumes/config/jitsi:/config:Z'
      - './volumes/data/jitsi/transcripts:/usr/share/jitsi-meet/transcripts:Z'
    networks:
      - jitsi

  prosody:
    image: jitsi/prosody:latest
    container_name: jitsi-prosody
    restart: always
    logging: *default-logging
    volumes:
      - './volumes/config/prosody:/config:Z'
      - './volumes/data/prosody:/prosody-plugins-custom:Z'
    networks:
      - jitsi

  jicofo:
    image: jitsi/jicofo:latest
    container_name: jitsi-jicofo
    restart: always
    logging: *default-logging
    volumes:
      - './volumes/config/jicofo:/config:Z'
    networks:
      - jitsi

  jvb:
    image: jitsi/jvb:latest
    container_name: jitsi-jvb
    restart: always
    logging: *default-logging
    volumes:
      - './volumes/config/jvb:/config:Z'
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
Primary:     ./volumes/{type}/{projectname}
Sub-Service: ./volumes/{type}/{servicename}
Database:    ./volumes/db/{dbtype}/{projectname}
Media:       ./volumes/data/media/{mediatype}
Downloads:   ./volumes/data/downloads
Logs:        ./volumes/data/log/{servicename}
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
