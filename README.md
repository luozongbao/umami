# Umami — Self-Hosted Web Analytics

Privacy-first web analytics, deployed with Docker Compose alongside the
existing production site (`wwwbanrimkwaecom_brk-network`).

---

## Table of contents

1. [Why this layout](#why-this-layout)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Quick start](#quick-start)
5. [Reverse-proxying with the existing nginx](#reverse-proxying-with-the-existing-nginx)
6. [Embedding the tracker on a website](#embedding-the-tracker-on-a-website)
7. [Day-2 operations](#day-2-operations)
8. [Backups & restores](#backups--restores)
9. [Upgrades](#upgrades)
10. [Troubleshooting](#troubleshooting)

---

## Why this layout

| Constraint on this server | What we did |
| --- | --- |
| UFW blocks most inbound ports (default deny). | Umami is **not** published to the host. Only the existing nginx on ports 80/443 is public. |
| Host ports 80/443 already used by the existing nginx. | Umami listens only on the internal docker network; nginx reverse-proxies `/umami/*` to it. |
| An existing MariaDB (`db` container) is used by the WordPress site. | Umami gets its **own** PostgreSQL container so the two apps cannot affect each other. |
| Cloudflare fronts the existing domain. | `CLIENT_IP_HEADER=X-Forwarded-For` and `set_real_ip_from` in nginx preserve the real visitor IP. |

> **Note on the database choice** — the official Umami Docker image only
> ships with the PostgreSQL driver (`DATABASE_TYPE=postgresql`). Using
> MariaDB/MySQL would require building a custom image. PostgreSQL is the
> officially supported database, has zero compatibility quirks, and is
> already what Umami is tested against.

---

## Architecture

```
Internet
   │
   ▼
Cloudflare
   │
   ▼
nginx  (host ports 80/443, container "nginx")
   │   location /umami/  ──►  http://umami:3000/
   │   (also serves your existing www.banrimkwae.com)
   ▼
┌──────────────────────────────────────────────────────────────┐
│ Docker network: wwwbanrimkwaecom_brk-network (external)      │
│                                                              │
│   ┌─────────────┐         ┌──────────────────────────────┐  │
│   │   umami     │ ──────► │          umami-db            │  │
│   │  :3000      │         │   postgres:16-alpine         │  │
│   └─────────────┘         │   volume: umami-db-data      │  │
│                           └──────────────────────────────┘  │
│                  on network: umami-net (private)            │
└──────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Docker Engine ≥ 24 and `docker compose` v2.
- Ability to edit the existing nginx container's config (for the reverse-proxy block).
- The existing `wwwbanrimkwaecom_brk-network` docker network must exist (it already does on this host).

---

## Quick start

```bash
# 1. Clone this repo onto the server (or copy the files into ~/umami).
cd ~/umami

# 2. Create the env file and generate secrets.
cp .env.example .env
chmod 600 .env
# Generate secrets (run each, paste the output into .env):
openssl rand -hex 32        # → UMAMI_APP_SECRET
openssl rand -hex 32        # → UMAMI_TWO_FACTOR_ENCRYPTION_KEY (must be 64 hex chars)
openssl rand -hex 24        # → UMAMI_DB_PASSWORD

# 3. Edit .env so the three values above are real secrets.
$EDITOR .env

# 4. Start the stack.
docker compose up -d

# 5. Check that both containers are healthy.
docker compose ps
docker compose logs --tail=50 umami
```

On first start, Umami automatically creates its tables inside
`umami-db` and seeds an admin account.

**Default login:** `admin` / `umami` — **change the password immediately** after the first login (top-right user menu → *Settings* → *Password*).

---

## Reverse-proxying with the existing nginx

The existing `nginx` container serves `www.banrimkwae.com` and
`banrimkwae.com`. Add a new location block to its config so traffic
for `/umami/` (and `/umami-api/` if you choose) is forwarded to the
Umami container on the docker network.

This repo assumes you serve Umami under the **`/umami/`** subpath — that
matches `BASE_PATH=/umami` already set in `docker-compose.yml`. If you
prefer a dedicated subdomain (e.g. `analytics.example.com`), drop the
`BASE_PATH` from `docker-compose.yml` and remove the `/umami/` prefix
from the nginx snippet.

### Snippet for the existing nginx config

```nginx
# /etc/nginx/conf.d/default.conf (inside the existing server { } block)

# ---- Umami analytics ----
location /umami/ {
    proxy_pass         http://umami:3000/;          # "umami" is the container hostname
    proxy_http_version 1.1;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   X-Forwarded-Host  $host;
    proxy_set_header   X-Forwarded-Port  $server_port;

    # Long-lived socket for the live dashboard.
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;

    # The Umami image's internal Next.js server needs the original
    # Host so the BASE_PATH redirect works correctly.
    proxy_redirect     off;
}

# The Umami tracker script (loaded by visitors' browsers).
location = /umami/script.js {
    proxy_pass http://umami:3000/script.js;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    add_header Cache-Control "public, max-age=3600";
}

# Heartbeat / static assets that don't go through Next.js rewrites.
location ~ ^/umami/(api|_next|static|favicons|images)/ {
    proxy_pass http://umami:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

After editing, reload nginx **inside its container**:

```bash
docker exec nginx nginx -t            # validate config
docker exec nginx nginx -s reload     # apply
```

> **Where the snippet goes:** the snippet above belongs inside the
> existing `server { listen 80; server_name www.banrimkwae.com ...; }`
> block of `/etc/nginx/conf.d/default.conf` inside the `nginx`
> container. If you have multiple `server { }` blocks (e.g. one per
> site), put it in the one that matches `www.banrimkwae.com`.

### Trusting Cloudflare's real client IP (already in your existing config)

Your existing nginx config already has `set_real_ip_from` lines for
the Cloudflare IP ranges and `real_ip_header CF-Connecting-IP;`. No
changes are needed — Umami will receive the visitor's true IP in
`X-Forwarded-For`.

---

## Embedding the tracker on a website

1. Log in to Umami at `https://www.banrimkwae.com/umami/`.
2. *Settings* → *Websites* → *Add website*.
3. Copy the **Tracker code** Umami gives you. For a site served over
   HTTPS, it looks like:

   ```html
   <script async defer
           data-website-id="YOUR-WEBSITE-ID"
           src="https://www.banrimkwae.com/umami/script.js"></script>
   ```

4. Paste it into the `<head>` of every page you want to track.

### Tracking multiple sites

Add a new website in Umami for each domain and use its own
`data-website-id`. The tracker script is shared.

---

## Day-2 operations

```bash
# View live logs
docker compose logs -f umami
docker compose logs -f umami-db

# Restart just Umami (after pulling a new image)
docker compose up -d --force-recreate umami

# Open a shell in the umami container
docker compose exec umami sh

# Open psql in the database container
docker compose exec umami-db psql -U umami -d umami

# Check disk usage of the DB volume
docker system df -v | grep umami-db-data
```

---

## Backups & restores

The only stateful component is the `umami-db-data` volume. Back it up
with `pg_dump` so you get a logical, portable snapshot (works even if
the postgres major version changes later).

```bash
# Backup (creates ./backups/umami-YYYYMMDD-HHMMSS.sql.gz)
mkdir -p backups
docker compose exec -T umami-db \
    pg_dump -U umami -d umami --no-owner --clean --if-exists \
    | gzip > "backups/umami-$(date +%Y%m%d-%H%M%S).sql.gz"

# Restore into a *stopped* stack
docker compose down
gunzip -c backups/umami-XXXXXXXX.sql.gz \
    | docker compose exec -T umami-db psql -U umami -d umami
docker compose up -d
```

> Backups contain visitor data and may include personal data subject
> to your privacy policy / GDPR — encrypt them at rest (e.g.
> `gpg --symmetric` before off-host upload).

---

## Upgrades

```bash
cd ~/umami
docker compose pull                 # fetches new umami + postgres tags
docker compose up -d --force-recreate   # recreates containers with the new images
docker image prune -f               # optional: reclaim space from old images
```

The `umami-db-data` volume is preserved across recreates — your data
is safe.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `docker compose up` says network `wwwbanrimkwaecom_brk-network` not found | Compose file declares it `external: true` but the network is missing. | `docker network create wwwbanrimkwaecom_brk-network` (or restore the network from your site setup). |
| `Umami login page 502` after editing nginx | Snippet is in the wrong `server { }` block, or `nginx -s reload` was skipped. | `docker exec nginx nginx -t`, fix the path, then reload. |
| All visitors show the same IP (e.g. your server's IP) | `real_ip_header` / `set_real_ip_from` lines for Cloudflare are missing in nginx. | Make sure the existing Cloudflare trust list is **above** the new `/umami/` block. |
| Umami can't connect to DB on first boot | `.env` was edited after the first run; password mismatch. | `docker compose logs umami-db` then `docker compose down -v && docker compose up -d` (⚠️ wipes DB). |
| `BASE_PATH` redirects break | The `BASE_PATH` in `docker-compose.yml` does not match the nginx `location`. | Keep them in sync (`/umami` here). |
| `pg_isready` healthcheck fails after restore | Bad dump or wrong DB user. | `docker compose exec umami-db psql -U umami -d umami -c '\dt'`. |

---

## Files in this repo

| File | Purpose |
| --- | --- |
| `docker-compose.yml` | Umami app + Postgres DB, attached to the existing brk-network. |
| `.env.example` | Template — copy to `.env` and fill in real secrets. |
| `.gitignore` / `.dockerignore` | Keep secrets and noise out of git and out of build context. |
| `RELEASE.md` | Notes for each deployed version. |

## License

Umami itself is MIT — see https://github.com/umami-software/umami.
