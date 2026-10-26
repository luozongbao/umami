# Project Guidelines

## Scope

This repository manages a production Umami deployment; it does **not** contain Umami application source. Treat upstream code, database migrations, and generated assets as image-owned. For operational procedures, read [README.md](README.md); for release-note conventions, read [RELEASE.md](RELEASE.md).

## Repository Layout

| Path | Purpose |
| --- | --- |
| [docker-compose.yml](docker-compose.yml) | Umami and PostgreSQL services, networking, healthchecks, and shared defaults. |
| [.env.example](.env.example) | Tracked template for deployment configuration; `.env` is local and secret. |
| [README.md](README.md) | Hosting patterns, nginx integration, tracker setup, operations, backup/restore, and troubleshooting. |
| [RELEASE.md](RELEASE.md) | Deployment history and change rationale. |

## Configuration Conventions

- Reuse the `x-umami-defaults` YAML anchor for service-level defaults; do not duplicate restart, init, or logging settings.
- When adding an environment variable, update [.env.example](.env.example), the relevant `environment:` mapping in [docker-compose.yml](docker-compose.yml), and any affected README instructions together.
- Preserve `.env` as a local secret. Never commit it or expose its values; `.env.example` may contain only placeholders.
- Keep `TZ=UTC` and `PGTZ=UTC`. Event timestamps depend on UTC.
- Keep Umami internal-only: publish no host port. Public access must continue through the existing nginx container.
- Preserve the external network semantics: compose key `brk-network`, actual external network name `app-network`.
- Keep the `umami-db-data` volume intact. `docker compose down -v` destroys the analytics database and requires explicit user approval.

## Runtime Invariants

- `UMAMI_BASE_PATH` must match the nginx `location` prefix. Use one hosting pattern per container: `/umami` for a sub-path or `/` for a dedicated subdomain.
- A custom `UMAMI_TRACKER_SCRIPT_NAME` must match every embedded tracker URL and every required nginx location.
- Under Pattern A, Cloudflare `set_real_ip_from` / `real_ip_header` directives must remain above the Umami location block.
- Changing `BASE_PATH`, secrets, or tracker configuration requires recreating the Umami container.

## Validation and Operations

Run commands from the repository root. There is no build, lint, typecheck, or automated test suite.

```bash
docker compose config
docker compose up -d --force-recreate umami
docker compose ps
docker compose logs --tail=50 umami
```

- Validate nginx changes inside its container: `docker exec nginx nginx -t`.
- Apply validated nginx changes with: `docker exec nginx nginx -s reload`.
- Use `docker compose exec umami-db pg_dump ...` for backups; follow the restore procedure in [README.md](README.md) exactly.

## Change Workflow

1. Make the smallest scoped configuration or documentation change.
2. Update [README.md](README.md) when behavior, nginx snippets, or the file inventory changes.
3. Add a new top entry to [RELEASE.md](RELEASE.md) using its **Added**, **Changed**, **Backwards compatibility**, and **Known limitations** structure when appropriate.
4. Run `docker compose config` and any relevant container-level validation.
5. Do not apply production changes or recreate containers unless the user explicitly requests it.
