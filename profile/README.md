## 👋 Welcome to composemgr 🚀

**135 production-ready, fully-documented Docker Compose projects for self-hosting**

All of these docker compose files use standardized environment settings.  
See the [main README](../../README.md) for complete documentation on environment variables and configuration.

## Quick Start

```bash
# Install any project
composemgr install postgres

# Or deploy directly
curl -q -LSsf "https://raw.githubusercontent.com/composemgr/postgres/main/docker-compose.yaml" | docker compose -f - up -d
```

## Documentation

- **[Main README](../../README.md)** - Complete guide with all environment variables
- **[Technical Specification](../SPEC.md)** - Detailed technical specification

## Categories (135 Projects)

- **Infrastructure** (18): postgres, mariadb, redis, nginx, portainer
- **Development** (12): gitlab, code-server, jenkins, registry
- **Media** (15): airsonic, ampache, arrstack, navidrome
- **Communication** (8): mattermost, jitsi, rocketchat, matrix
- **Productivity** (18): nextcloud, affine, docmost, blinko
- **Auth & Security** (5): authentik, keycloak, vaultwarden
- **And 59 more...**

## Author

🤖 casjay: [Github](https://github.com/casjay) 🤖  
🦄 composemgr: [Github](https://github.com/composemgr) 🦄
