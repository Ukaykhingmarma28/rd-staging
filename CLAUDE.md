# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Docker Compose stack for a WordPress + WooCommerce site. Runs behind Cloudflare (handles SSL termination) with Traefik as the ingress router on the host.

## Architecture

```
Traefik (host) → Nginx (reverse proxy) → WordPress (PHP-FPM) → MariaDB
```

- All services run on a Docker overlay network (`wordpress_net`). Nginx does not expose ports directly — Traefik routes to it via the overlay network.
- WordPress content is in the `wordpress_data` named volume, shared read-write with WordPress and read-only with Nginx.
- **Startup ordering**: WordPress waits for MariaDB healthcheck to pass before starting. Nginx waits for WordPress.

### Services & Resource Limits

| Service   | Image                      | CPU Limit | RAM Limit | Key Tuning                                    |
|-----------|----------------------------|-----------|-----------|-----------------------------------------------|
| mariadb   | mariadb:11.8               | 1.5       | 1536M     | InnoDB buffer pool 1024M, 100 max connections |
| wordpress | wordpress:6.9-fpm-alpine   | 2         | 2304M     | PHP-FPM dynamic, 8 max children, JIT enabled  |
| nginx     | nginx:1.28-alpine          | 0.5       | 256M      | Tiered bot rate limiting, gzip                |

## Key Configuration Files

- `docker-compose.yml` — Service definitions, volumes, networks, resource limits
- `.env.example` — Required env vars: DB creds, `DOMAIN`, `WORDPRESS_TABLE_PREFIX`
- `nginx/nginx.conf` — Worker tuning (auto CPUs, 1024 connections), gzip, server_tokens off
- `nginx/default.conf` — Server block: 3-tier bot rate limiting, scraper query blocking, security headers, static asset caching, Cloudflare HTTPS detection via `CF-Visitor`
- `php/custom.ini` — PHP limits (256M memory, 128M uploads), OPcache (128M, `validate_timestamps=0`), JIT (64M buffer, mode 1255)
- `php/www.conf` — PHP-FPM pool: dynamic PM, 8 max children, 500 max_requests, slow log at 5s
- `mariadb/custom.cnf` — InnoDB 1024M buffer pool, 128M log files, slow query log (>1s), utf8mb4

All config files are mounted read-only (`:ro`). Changes require a container restart/recreate, not just a service reload.

## Common Commands

```bash
# Start / stop
docker compose up -d
docker compose down

# Logs
docker compose logs -f [service]

# Restart a single service
docker compose restart nginx

# Apply config changes (recreates only changed services)
docker compose up -d --force-recreate [service]

# Container shells
docker compose exec wordpress sh
docker compose exec mariadb mysql -u root -p

# Check PHP-FPM status (from within the network)
docker compose exec nginx curl http://wordpress:9000/status
```

## Important Design Decisions

- **SSL is handled by Cloudflare**, not this stack. Nginx listens on port 80 only. The `CF-Visitor` header is mapped to `$cf_https` and passed as `HTTPS` fastcgi_param so WordPress generates correct URLs.
- **No FastCGI cache** — was removed. Caching relies on Cloudflare edge.
- **OPcache `validate_timestamps=0`** — PHP never checks if files changed. After deploying code/plugin updates, restart the WordPress container.
- **JIT is enabled** (mode 1255, 128M buffer) for uncached WooCommerce request speedup.
- **3-tier bot rate limiting** in `nginx/default.conf`:
  - Tier 1 (SEO bots: Google, Bing, social previews): 2r/s with burst=5
  - Tier 2 (SEO tools, AI bots: Ahrefs, GPTBot, ClaudeBot): 30r/m with burst=3
  - Tier 3 (generic scrapers: curl, wget, python-requests, empty UA): blocked 403
  - Scraper query patterns (`per_row=`, `filter_*` taxonomy params, price+sort combos, `shop_view=` combos): blocked 429
- **General per-IP PHP rate limit**: 10r/s with burst=30
- **wp-login.php has a 2-hour `fastcgi_read_timeout`** (7200s) — separate from the standard 300s timeout on other PHP locations.
- **Traefik integration** via the overlay network. The `Host` rule uses the `DOMAIN` env var (Traefik labels are applied externally, not in this compose file).
- **MariaDB `MARIADB_DISABLE_IO_URING=1`** — workaround for io_uring issues in containerized environments.
- The network uses `overlay` driver with `attachable: true` (Docker Swarm compatible, works on single-host too).
