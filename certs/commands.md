# Certificates (local + Let's Encrypt)

This repo expects Nginx to load certificates from:

- `./cert/mattermost.crt`
- `./cert/mattermost.key`

Nginx mounts `./cert` to `/etc/nginx/ssl` (read-only) via `docker-compose.yml`.

> Tip: use a URL-safe database password in `.env` (or URL-encode it inside `MM_SQL_DATASOURCE`).

---

## A) Local / lab (self-signed)

1) Create the cert/key:

```bash
mkdir -p cert
DOMAIN="chat.example.com"

openssl req -x509 -nodes -newkey rsa:4096 -days 825   -keyout cert/mattermost.key   -out cert/mattermost.crt   -subj "/CN=${DOMAIN}"   -addext "subjectAltName=DNS:${DOMAIN}"
```

2) Restart Nginx:

```bash
docker compose restart nginx
```

> For iOS/macOS/Windows clients, you must trust the self-signed cert (or better: use an internal CA and sign the cert).

---

## B) Let's Encrypt (HTTP-01)

Requirements:
- `DOMAIN` must resolve to this server public IP.
- Port **80/TCP** must be reachable from the Internet (LetsEncrypt validation).
- This is **not** suitable for fully air-gapped networks.

1) Create required folders:

```bash
mkdir -p certbot/www certbot/conf cert
DOMAIN="chat.example.com"
EMAIL="admin@example.com"
```

2) Issue a certificate using the webroot method:

```bash
docker run --rm -it   -v "$(pwd)/certbot/www:/var/www/certbot"   -v "$(pwd)/certbot/conf:/etc/letsencrypt"   certbot/certbot:latest certonly   --webroot -w /var/www/certbot   -d "${DOMAIN}"   --email "${EMAIL}"   --agree-tos --no-eff-email
```

3) Copy the issued cert/key to the paths used by Nginx in this repo:

```bash
cp "certbot/conf/live/${DOMAIN}/fullchain.pem" cert/mattermost.crt
cp "certbot/conf/live/${DOMAIN}/privkey.pem" cert/mattermost.key
chmod 600 cert/mattermost.key
```

4) Reload Nginx:

```bash
docker compose exec nginx nginx -s reload || docker compose restart nginx
```

### Renew (cron/systemd timer)

LetsEncrypt certs expire quickly; renew periodically:

```bash
docker run --rm   -v "$(pwd)/certbot/www:/var/www/certbot"   -v "$(pwd)/certbot/conf:/etc/letsencrypt"   certbot/certbot:latest renew   --webroot -w /var/www/certbot
```

After renewal, re-copy certs and reload Nginx:

```bash
DOMAIN="chat.example.com"
cp "certbot/conf/live/${DOMAIN}/fullchain.pem" cert/mattermost.crt
cp "certbot/conf/live/${DOMAIN}/privkey.pem" cert/mattermost.key
docker compose exec nginx nginx -s reload || docker compose restart nginx
```

---

## Notes

- If you prefer not to copy certs, you can point Nginx directly to:
  `/etc/letsencrypt/live/<DOMAIN>/{fullchain.pem,privkey.pem}` and keep the
  `./certbot/conf:/etc/letsencrypt:ro` mount (already present in compose).
