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
| Host ports 80/443 already used by the existing nginx. | Umami listens only on the internal docker network; nginx reverse-proxies to it. |
| An existing MariaDB (`db` container) is used by the WordPress site. | Umami gets its **own** PostgreSQL container so the two apps cannot affect each other. |
| Cloudflare fronts the existing domain. | `CLIENT_IP_HEADER=X-Forwarded-For` and `set_real_ip_from` in nginx preserve the real visitor IP. |
| Same Umami instance serves analytics for **multiple other websites**. | One Umami container tracks any number of sites (configured in the Umami UI). The dashboard itself can also be exposed under multiple hostnames via nginx vhosts — see [Hosting patterns](#hosting-patterns). |

> **Note on the database choice** — the official Umami Docker image only
> ships with the PostgreSQL driver (`DATABASE_TYPE=postgresql`). Using
> MariaDB/MySQL would require building a custom image. PostgreSQL is the
> officially supported database, has zero compatibility quirks, and is
> already what Umami is tested against.

---

## Hosting patterns

You can reuse this single Umami instance for as many websites as you want.
There are **two layers** of "many":

1. **Tracking many sites** (always supported) — every site you want to
   track is added inside Umami's dashboard, then you paste its tracker
   `<script>` into that site. See [Embedding the tracker on a website](#embedding-the-tracker-on-a-website).
2. **Exposing the Umami dashboard under many hostnames** (this section) —
   pick **one** of the two patterns below. Both serve the same Umami
   container; only the nginx vhost changes.

| Pattern | `UMAMI_BASE_PATH` | Public URL | Best when |
| --- | --- | --- | --- |
| **A — Same domain, sub-path** *(default)* | `/umami` | `https://www.banrimkwae.com/umami/` | Tracked sites and dashboard share a domain (simplest, no third-party cookies). |
| **B — Dedicated subdomain** | `/` | `https://analytics.example.com/` | Tracked sites live on **different** domains and you want one central analytics URL. |

> **Pick one pattern.** Mixing `/umami` on one vhost and `/` on another
> against the same Umami container breaks the `BASE_PATH` invariant —
> stick to one pattern per container.

### Pattern A — Same domain, sub-path (default)

Already wired up by `UMAMI_BASE_PATH=/umami` in `.env.example`. Drop the
[snippet A](#snippet-for-pattern-a--same-domain-sub-path) into the
existing `server { }` block for each domain that should host the
dashboard. Every site you add in Umami's UI can still live on its own
domain — only the **dashboard URL** is path-based.

### Pattern B — Dedicated subdomain

1. In `.env`, change `UMAMI_BASE_PATH=/` and recreate the umami container:
   ```bash
   sed -i 's|^UMAMI_BASE_PATH=.*|UMAMI_BASE_PATH=/|' .env
   docker compose up -d --force-recreate umami
   ```
2. Add a new vhost in nginx for `analytics.example.com` (point DNS at
   the same server, see [snippet B](#snippet-for-pattern-b--dedicated-subdomain)).
3. Tracker scripts embedded on every tracked site now point at the
   analytics subdomain, e.g.
   `<script src="https://analytics.example.com/script.js" …></script>`.
   The `data-website-id` still identifies each site.

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
`banrimkwae.com`. Add **one of the two snippets below** to expose the
Umami dashboard.

> Both snippets assume nginx can resolve the docker hostname
> `umami` — it does, because the `nginx` container is on
> `wwwbanrimkwaecom_brk-network`, which our `docker-compose.yml`
> joins as an external network.

### Snippet for Pattern A — same domain, sub-path

Drop into **every** `server { }` block that should host the dashboard
(typically just the `www.banrimkwae.com` vhost).

```nginx
# /etc/nginx/conf.d/default.conf (inside the server { } block for www.banrimkwae.com)

# ---- Umami analytics (sub-path) ----
# BASE_PATH=/umami  in .env  ⇒  dashboard at https://<host>/umami/
location /umami/ {
    proxy_pass         http://umami:3000/;
    proxy_http_version 1.1;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   X-Forwarded-Host  $host;
    proxy_set_header   X-Forwarded-Port  $server_port;

    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
    proxy_redirect     off;
}

# Tracker script (loaded by visitors of every tracked site).
# If you set UMAMI_TRACKER_SCRIPT_NAME=stats.js, change both URLs below.
location = /umami/script.js {
    proxy_pass http://umami:3000/script.js;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    add_header Cache-Control "public, max-age=3600";
}

# Static assets & API routes — Next.js rewrites.
location ~ ^/umami/(api|_next|static|favicons|images)/ {
    proxy_pass http://umami:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

To expose the dashboard on **multiple existing domains**, repeat the
three blocks inside each vhost (e.g. one for `www.banrimkwae.com`,
one for `banrimkwae.com`). The Umami container itself doesn't care
which host served the request — only the browser's tracker script
needs to load from the same domain as the tracked page.

### Snippet for Pattern B — dedicated subdomain

1. Point `analytics.example.com` DNS at this server (A/AAAA record).
2. Set `UMAMI_BASE_PATH=/` in `.env` and recreate the umami container:
   ```bash
   sed -i 's|^UMAMI_BASE_PATH=.*|UMAMI_BASE_PATH=/|' .env
   docker compose up -d --force-recreate umami
   ```
3. Add a new file in `/etc/nginx/conf.d/` (e.g. `umami.conf`):

```nginx
# /etc/nginx/conf.d/umami.conf
# Dedicated subdomain vhost for the Umami dashboard.
server {
    listen      80;
    listen [::]:80;
    server_name analytics.example.com;

    # Reuse the same Cloudflare trust block you already have elsewhere
    # (set_real_ip_from / real_ip_header CF-Connecting-IP).
    # … paste the same lines here …

    location / {
        proxy_pass         http://umami:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;
        proxy_set_header   X-Forwarded-Port  $server_port;

        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
        proxy_redirect     off;
    }
}
```

Add an HTTPS server block the same way (or use a separate file with
`listen 443 ssl http2;` plus your existing cert paths) once a cert is
issued for `analytics.example.com`.

### Trusting Cloudflare's real client IP (already in your existing config)

Your existing nginx config already has `set_real_ip_from` lines for
the Cloudflare IP ranges and `real_ip_header CF-Connecting-IP;`. Copy
those lines into the new Pattern B vhost (snippet above) so visitor
IPs are preserved there too.

After editing any nginx file, validate and reload **inside** the
nginx container:

```bash
docker exec nginx nginx -t            # validate config
docker exec nginx nginx -s reload     # apply
```

---

## Embedding the tracker on a website

1. Log in to Umami (Pattern A: `https://www.banrimkwae.com/umami/` —
   Pattern B: `https://analytics.example.com/`).
2. *Settings* → *Websites* → *Add website*.
3. Copy the **Tracker code** Umami gives you.

   **Pattern A — sub-path on the same domain as the tracked site:**
   ```html
   <script async defer
           data-website-id="YOUR-WEBSITE-ID"
           src="https://www.banrimkwae.com/umami/script.js"></script>
   ```

   **Pattern B — dedicated analytics subdomain** (works for tracked
   sites on *any* domain):
   ```html
   <script async defer
           data-website-id="YOUR-WEBSITE-ID"
           src="https://analytics.example.com/script.js"></script>
   ```

4. Paste it into the `<head>` of every page you want to track.

### Tracking many sites from one Umami instance

A single Umami container can collect analytics for **any number of
websites**, on any number of domains. The procedure is identical for
each:

1. *Settings* → *Websites* → *Add website* (give it a name + the
   site's domain).
2. Copy the tracker `<script>` Umami generates — only the
   `data-website-id` differs per site.
3. Paste it into the target site's `<head>`.
4. (Optional) Create **teams** under *Settings* → *Teams* so different
   clients only see their own site's stats.

> **CORS / ad-blocker note:** Pattern A keeps the tracker on the same
> origin as the tracked site, so no CORS configuration is needed.
> Pattern B requires the analytics subdomain to send
> `Access-Control-Allow-Origin: *` for the `/api/send` endpoint;
> Umami's `ghcr.io/umami-software/umami:postgresql-latest` image
> already sets this header, so no extra config is required.
> If you renamed the script via `UMAMI_TRACKER_SCRIPT_NAME`, see the
> `UMAMI_TRACKER_SCRIPT_NAME` note below.

### Renaming the tracker script (optional)

Ad-blockers often block `script.js`. To dodge that, set in `.env`:

```
UMAMI_TRACKER_SCRIPT_NAME=stats.js
```

Then update every tracker `<script src="…">` on every tracked site
from `…/script.js` to `…/stats.js`, **and** if you're on Pattern A,
add one more nginx location inside each vhost that hosts the
dashboard:

```nginx
location = /umami/stats.js {
    proxy_pass http://umami:3000/stats.js;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    add_header Cache-Control "public, max-age=3600";
}
```

Pattern B requires no nginx change because the generic `location /`
already proxies everything.

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
| `docker-compose.yml` | Umami app + Postgres DB, attached to the existing brk-network. `BASE_PATH`, `TRACKER_SCRIPT_NAME` and `ALLOWED_FRAME_URLS` are env-driven so the same compose file serves any hosting pattern. |
| `.env.example` | Template — copy to `.env` and fill in real secrets and the chosen hosting-pattern knobs. |
| `.gitignore` / `.dockerignore` | Keep secrets and noise out of git and out of build context. |
| `RELEASE.md` | Notes for each deployed version. |

## License

Umami itself is MIT — see https://github.com/umami-software/umami.
