# AI assistant guide for this repo (linkstack-docker)

This repo packages LinkStack (a PHP app) into a Docker image using Alpine Linux, Apache HTTPD, and PHP 8.2. The image serves the app from `/htdocs` and exposes HTTP (80) and HTTPS (443).

## Architecture and runtime
- Base/image: `alpine:3.22` with `apache2`, `apache2-ssl`, `php82-apache2` and common PHP 8.2 extensions (pdo_mysql/pgsql/sqlite, gd, mbstring, zip, redis, etc.). See `Dockerfile` for the full list.
- App payload: The `linkstack/` folder (empty in git except `.gitignore`) is copied verbatim to `/htdocs` at build time. Drop upstream LinkStack release files here before building.
- Web server: Apache configs originate in `configs/apache2/httpd.conf` and `configs/apache2/ssl.conf` and are copied into `/etc/apache2` during build; logs go to stdout/stderr.
- TLS: Default cert/key paths: `/etc/ssl/apache2/server.pem` and `/etc/ssl/apache2/server.key` (from `apache2-ssl`). Mount real certs there in production or edit `ssl.conf`.
- User/permissions: Container runs as `apache:apache` (see `Dockerfile` USER). Ensure mounted volumes grant write rights to this user.
- Healthcheck: `HEALTHCHECK` curls `http://localhost` with `-A HealthCheck`.

## Data and persistence
- Persist `/htdocs` to retain config, uploads, themes, and DB (for SQLite). Guidance in README:
  - Important paths: `/htdocs/.env`, `/htdocs/database/database.sqlite`, `/htdocs/config/advanced-config.php`, `/htdocs/themes`, `/htdocs/assets/img`, `/htdocs/assets/linkstack/images/avatar.png`.
- MySQL option: `docker-compose.yml` includes a `mysql:8` service; configure LinkStack DB via `/htdocs/.env`.

## Configuration inputs
- Env vars surfaced into Apache via `PassEnv` in `httpd.conf`/`ssl.conf`:
  - `SERVER_ADMIN`, `HTTP_SERVER_NAME`, `HTTPS_SERVER_NAME`, `LOG_LEVEL`.
- Other runtime envs (handled by entrypoint/OS): `TZ`, `PHP_MEMORY_LIMIT`, `UPLOAD_MAX_FILESIZE` (see README’s Deployment section).
- PHP ini: `configs/php/php.ini` is copied to `/etc/php8.2/php.ini` at build. It’s empty by default—add overrides here if needed.

## Developer workflows (actionable)
- Build a custom image (requires upstream payload in `linkstack/`):
  1) Download latest LinkStack release zip; extract directly into `linkstack/`.
  2) Build: `docker build -t linkstack .`
- Run with persistent storage:
  - `docker volume create linkstack_data`
  - `docker run -d --name linkstack -p 80:80 -p 443:443 --mount source=linkstack_data,target=/htdocs --restart unless-stopped linkstackorg/linkstack`
- Compose usage:
  - README shows a preferred compose with `linkstack_data:/htdocs` and HTTPS on the reverse proxy.
  - Repo `docker-compose.yml` includes MySQL and maps `8080:80` and `8081:443`; adjust volumes to a named volume for persistence.

## Conventions and gotchas
- Entry banner: `docker-entrypoint.sh` reads `/htdocs/version.json` and the script uses `set -eu`. If that file is missing in the payload, the container will exit—ensure releases include it.
- PID handling: entrypoint removes stale `/htdocs/httpd.pid` before starting `httpd -D FOREGROUND`.
- Reverse proxy: Use HTTPS end-to-end to avoid mixed content. README includes an NGINX example (with websocket upgrade headers and a CSP `upgrade-insecure-requests`).

## Edit map
- Apache: `configs/apache2/httpd.conf`, `configs/apache2/ssl.conf`.
- PHP: `configs/php/php.ini` (copied to `/etc/php8.2/php.ini`).
- App files: provide upstream LinkStack into `linkstack/` before building; at runtime, persist `/htdocs`.

If your workflow differs (e.g., SQLite vs MySQL, different reverse proxy), tell me and I’ll tailor these directions accordingly.