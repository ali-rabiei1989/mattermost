# Mattermost (Docker Compose)

## Quick start

1. Copy environment file and set the values
```bash
cp .env.example .env
vim .env
```
2. Create directories
```bash
mkdir -p postgres mattermost/{config,data,logs,plugins,client/plugins} nginx/conf.d cert certbot/{www,conf}
```
3. create a local cert or LetsEncrypt cert (see cert/commands.md)
```
See cert/commands.md.
```
4. Run the compose file
```bash
docker compose up -d
```
---
- Nginx listens on **80/443**
- Mattermost is exposed internally on **8065** (not published)