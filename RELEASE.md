# Release notes

All notable changes to this deployment configuration are documented here.
Format loosely follows [Keep a Changelog](https://keepachangelog.com/);
versions follow the tracked Umami upstream version we pin to.

---

## [1.1.0] — 2026-08-30 — Multi-website / multi-domain support

### Added

- `UMAMI_BASE_PATH` env var (`/umami` by default) — choose between:
  - **Pattern A** `/umami` — sub-path on the same domain as a tracked site
    (e.g. `https://www.banrimkwae.com/umami/`).
  - **Pattern B** `/` — apex of a dedicated analytics subdomain
    (e.g. `https://analytics.example.com/`).
- `UMAMI_TRACKER_SCRIPT_NAME` env var — rename the tracker script
  (e.g. `stats.js`) to bypass ad-blockers; works with both hosting
  patterns.
- `UMAMI_ALLOWED_FRAME_URLS` env var — space-separated hostnames that
  may iframe the Umami dashboard (e.g. an admin panel hosted on
  another domain).
- README section **Hosting patterns** with full nginx snippets for
  both Pattern A and Pattern B.
- README guidance on adding many websites inside one Umami instance
  (per-site `data-website-id`, optional Teams for multi-tenant
  isolation).

### Changed

- `BASE_PATH` in `docker-compose.yml` is now env-driven
  (`${UMAMI_BASE_PATH:-/umami}`) instead of hard-coded.
- Tracker section in README rewritten to show both hosting patterns
  and explain how to rename the tracker script consistently across
  nginx + tracked sites.

### Backwards compatibility

- Existing deployments keep working without changes: the default
  `UMAMI_BASE_PATH=/umami` matches the previous hard-coded value.
- No DB migration needed; no volume changes.

### Known limitations

- One Umami container supports **one** `BASE_PATH` at a time. To
  serve `/umami` on one vhost and `/` on another against the same
  container would break Next.js's internal redirects — pick one
  pattern per deployment.
- Renaming the tracker script (`UMAMI_TRACKER_SCRIPT_NAME`) under
  Pattern A requires an extra `location = /umami/<name>` block per
  vhost; Pattern B requires none.

---

## [1.0.0] — 2026-08-30 — Initial production deployment

### Added

- `docker-compose.yml` running:
  - `umami` from `ghcr.io/umami-software/umami:postgresql-latest`
  - `umami-db` from `postgres:16-alpine`
- Internal-only networking — Umami is **not** published to host ports.
  Reachable only via the existing `nginx` container over the
  pre-existing `wwwbanrimkwaecom_brk-network` docker network
  (declared `external: true` in compose).
- `BASE_PATH=/umami` so Umami is served under the `/umami/` URL prefix
  on `www.banrimkwae.com` (no new public port needed; UFW-safe).
- `CLIENT_IP_HEADER=X-Forwarded-For` + nginx `set_real_ip_from`
  block preserve the real visitor IP behind Cloudflare.
- Healthchecks on both containers (`pg_isready`, `/api/heartbeat`).
- `init: true`, `no-new-privileges`, JSON-file log rotation
  (`max-size: 10m`, `max-file: 3`).
- Named volume `umami-db-data` for persistent Postgres storage.
- `.env.example` template documenting required env vars.
- `.gitignore` and `.dockerignore` to keep secrets and build noise
  out of git and Docker build contexts.
- README covering reverse-proxy snippet, tracker embed, backups,
  upgrades and troubleshooting.

### Design decisions

- **PostgreSQL instead of MariaDB/MySQL.** Umami's official image
  only ships the PostgreSQL driver (`DATABASE_TYPE=postgresql`).
  Adding MariaDB/MySQL support requires a custom image build, which
  adds maintenance burden for no functional benefit. PostgreSQL is
  the supported database upstream and is what we run.
- **Separate database container.** The existing site uses a shared
  MariaDB (`db` container) for WordPress; Umami gets its own
  isolated Postgres to keep blast radius minimal.
- **No host port for Umami.** Public traffic flows: internet →
  Cloudflare → existing nginx (80/443) → Umami (3000) on the
  docker network. UFW requires no new allow rules.

### Defaults

| Setting | Value | Notes |
| --- | --- | --- |
| `BASE_PATH` | `/umami` | Must match the nginx `location /umami/` prefix. |
| Umami port (in-container) | `3000` | Umami default. Not published. |
| Postgres major | `16` | Aligned with Umami's supported versions. |
| `TZ` / `PGTZ` | `UTC` | Required for correct event timestamps. |
| `DISABLE_TELEMETRY` | `1` | Fully self-hosted, no phone-home. |
| `DISABLE_UPDATES` | `1` | Upgrades are explicit `docker compose pull`. |

### Operational notes

- First boot creates the database schema and seeds the admin account
  (`admin` / `umami`). **Change the password immediately.**
- 2FA requires `UMAMI_TWO_FACTOR_ENCRYPTION_KEY` (64 hex chars);
  generate with `openssl rand -hex 32`.
- To rotate `UMAMI_APP_SECRET`: stop umami, change the value in
  `.env`, `docker compose up -d --force-recreate umami`. Existing
  login sessions are invalidated (users must log in again).

### Known limitations

- The compose file attaches to `wwwbanrimkwaecom_brk-network` via
  `external: true`. If you ever rebuild the host, recreate that
  network first or `docker compose up` will fail with
  `network ... not found`.
- Tracker `script.js` is served from the same `/umami/` path as the
  dashboard; some ad-blockers may block the default name. If that
  becomes an issue, set `TRACKER_SCRIPT_NAME=stats.js` in
  `docker-compose.yml` and update the `<script src=...>` on your
  site accordingly.
