# Production: Docker and deploy

Source: the `mini-crm` project (Laravel 13 + PHP 8.4 + Vue 3/Vite + MySQL 8.4), running in production on one 2 GB server
shared with other sites. This is the reference shape for every new Laravel project. All rules here are **[confirmed]**
unless marked otherwise. Ready files: [templates/deploy/](../templates/deploy/README.md).

The runtime inside the image is php-fpm. The code rules written for Octane (stateless `readonly` classes, no leaks) still
apply: the same code must run correctly on both.

---

## The non-negotiable rules

### 1. Images have no tags
`image: {{project}}`, `{{project}}-web`, `{{project}}-backup` in `compose.yaml` — never `:latest`, `:v1.2`, `:<sha>`.
Every deploy builds the images again from the current commit; the previous build loses its name to the new one and is
removed by `docker image prune -f --filter "label=project={{project}}"`. So **every stage** of the Dockerfile carries
`LABEL project={{project}}` (directly or through the stage it is built `FROM`).
*Why:* the commit is the version. A tag is a second version number that can disagree with the code; a registry of tagged
images is infrastructure nobody needs on one server. The label lets the prune touch only this project's images, never
other sites on the same Docker.

### 2. No CI/CD
Deploy is manual: `deploy/deploy.sh` is run on the server by hand. The author pushes by hand (the shell function `g`),
only to `main`. No GitHub Actions, no webhooks, no auto-deploy on push.
An assistant never pushes and never deploys; it prepares the commit, and the author runs `g` and `deploy.sh`.
*Why:* one person, one server — a pipeline adds secrets in a third-party service, a second place where deploy logic lives,
and deploys nobody watched. A deploy by hand is a deploy someone is looking at.

### 3. Server nginx: a copy of the repo file + `certbot --nginx`
The server's nginx config lives in the repository (`deploy/{{project}}.conf`), HTTP only, one `server` block per domain.
On the server it is a **copy**, not a symlink:
```bash
sudo cp deploy/{{project}}.conf /etc/nginx/sites-available/{{project}}.conf
sudo ln -s /etc/nginx/sites-available/{{project}}.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx --cert-name {{domain}} -d {{domain}} --redirect --hsts   # once per domain
```
After the repo file changes: copy it again, then `certbot --nginx --cert-name {{domain}} -d {{domain}} --reinstall --redirect --hsts`
for every domain (the fresh copy lost certbot's HTTPS lines).
*Why:* certbot writes HTTPS, the certificate paths and the redirect into the file. A symlink would point certbot's edits
into the git checkout, and the next `git reset --hard` in a deploy would wipe HTTPS. A copy is untouched by deploys.
One block and one certificate per domain: domains are independent, one expiring or being removed does not break another.

### 4. Rollback = rebuilding an older commit
- **The right way:** `git revert <commit>` in `main`, push, run `deploy.sh` again.
- **Urgent:** on the server `git reset --hard <good commit>`, then
  `docker compose build --pull && docker compose up -d --wait`. The next `deploy.sh` brings the server back to `main`.
- A rollback never runs `migrate:rollback` in production: a migration that must be undone is undone by a **new** migration.
*Why:* there are no old images to switch to (rule 1) — and none are needed: the commit fully defines the image.

### 5. Secrets live only in `.env` on the server
- `.env` reaches the containers through `env_file: .env` and **never** the image: `.env` and `.env.*` are in
  `.dockerignore` (except `!.env.example`).
- `DB_ROOT_PASSWORD` is given only to the `db` service; `app` gets it blanked: `environment: DB_ROOT_PASSWORD: ""`.
- The app connects as a non-root user (`DB_USERNAME` ≠ `root`); passwords are `openssl rand -hex 24`.
*Why:* an image can be copied, cached or pushed; anything baked into it leaks with it. The web process never needs the
root password, so a code-execution bug in the app cannot drop or read other databases.

### 6. Migrations after the first production run: only new ones
Deploy runs `php artisan migrate --force` — never `migrate:fresh` (that is for the very first launch only). An already
applied migration is never edited: a change is a new migration.
*Why:* `migrate` skips migrations already recorded in the `migrations` table, so an edited old one is silently never
applied in production, and the schema in production drifts from the code.

---

## Files in the project

```
Dockerfile                 one file, multi-stage, three final images
.dockerignore
compose.yaml               name: {{project}}
docker/php.ini             on top of php.ini-production
docker/php-fpm.conf
docker/entrypoint.sh       php artisan optimize on start
docker/nginx.conf          nginx inside the web container
docker/nginx-php.conf      the fastcgi snippet
docker/backup.sh           nightly mysqldump
deploy/deploy.sh           the manual deploy
deploy/{{project}}.conf    the server's nginx (copied, see rule 3)
deploy/README.md           first launch, rollback, backups, restore — for a human, in the author's language
```

---

## Dockerfile: one file, multi-stage

| Stage | Base | What it does | Why |
|---|---|---|---|
| `php` | `php:8.4-fpm-trixie` | extensions `intl`, `opcache`, `pdo_mysql`; `php.ini-production` → `php.ini`, then `docker/php.ini` as `conf.d/zz-{{project}}.ini` and `docker/php-fpm.conf` as `php-fpm.d/zz-{{project}}.conf` | `zz-` loads last and wins over defaults; only the extensions the app uses |
| `vendor` | `FROM php` + `COPY --from=composer:2` | only `composer.json`/`composer.lock`, `composer install --no-dev --no-scripts --no-autoloader --prefer-dist` | the dependency layer is cached separately from the code: a code change does not reinstall vendor |
| `assets` | `node:22-alpine` | `npm ci`, `npm run build` → `public/build` | node never reaches a final image |
| `app` (final, php-fpm) | `FROM php` | vendor + code + `public/build`; `composer dump-autoload --no-dev --optimize --classmap-authoritative`, `php artisan package:discover`, then **delete composer**; create the `storage/*` folders, owner `www-data`; `ENV HOME=/tmp`; `USER www-data`; `ENTRYPOINT entrypoint`, `CMD php-fpm` | authoritative classmap = no filesystem lookups for classes; no composer in production; not root |
| `web` (final) | `nginx:1.30-alpine` | `docker/nginx.conf`, the `nginx-php.conf` snippet, and **only** `public/` copied from `app` | nginx serves static files itself and never sees the PHP code |
| `backup` (final) | `mysql:8.4` — the same version as `db` | `docker/backup.sh` as the entrypoint | Debian's `mysqldump` is MariaDB's, and MySQL rejects its dumps on restore |

- **`ENV HOME=/tmp`** — `www-data`'s home (`/var/www`) is not writable, and `tinker` (psysh) writes its config and history
  there; without it `docker compose exec app php artisan tinker` fails.
- **The config cache is built on start, not in the build** (`docker/entrypoint.sh`: `php artisan optimize` when the
  command is `php-fpm`). *Why:* the cache freezes the values of `.env`, and `.env` exists only on the server, not at build
  time. `php artisan optimize` in the Dockerfile would bake an empty config into the image.
- **Base images are pinned to a minor line** (`php:8.4-…`, `nginx:1.30-…`, `mysql:8.4`, `node:22-…`, `composer:2`), and
  `build --pull` fetches their latest patch every deploy. These are *base* image tags from Docker Hub — rule 1 is about
  *our* images.

## `.dockerignore`

Excludes: `.env`, `.env.*` (keeps `!.env.example`), `.git`, `.github`, `.claude`, IDE folders, `node_modules`, `vendor`,
`deploy`, `tests` and test output, **`public/hot`**, `public/build`, `public/storage`, `bootstrap/cache/*.php`, and the
**contents** of `storage/*` (logs, sessions, views, cache, uploaded files).
*Why:* secrets never enter the build context; `vendor`/`node_modules`/`public/build` are produced inside the build, a local
copy would override it; a leftover `public/hot` (the Vite dev-server marker) makes the production page load its assets
from `localhost:5173`; local `bootstrap/cache/*.php` would carry a dev config and dev providers into production.

## `compose.yaml`

`name: {{project}}` at the top: volumes become `{{project}}_storage`, `{{project}}_db`, `{{project}}_backups`
(on the host `/var/lib/docker/volumes/{{project}}_*`), independent of the folder name.

- **`app`** — `image: {{project}}`, `build.target: app`, `env_file: .env`, `DB_ROOT_PASSWORD: ""`,
  volume `storage:/var/www/html/storage`, `depends_on: db: condition: service_healthy`.
- **`web`** — `image: {{project}}-web`, `build.target: web`, port **only on localhost**:
  `"127.0.0.1:${PORT:?PORT is not set}:80"`. Healthcheck `wget -qO /dev/null http://127.0.0.1/up`.
  *Why:* the server's nginx is the only way in (HTTPS lives there); a port on `0.0.0.0` would bypass HTTPS and the real-IP
  logic. `/up` goes through nginx → php-fpm → Laravel → a DB query, so "healthy" means the whole request path works,
  not just that nginx is up. `:?` makes compose refuse to start with an unset `PORT` instead of picking a default.
- **`db`** — `image: mysql:8.4`, no build. Volume `db:/var/lib/mysql`. Healthcheck `mysqladmin ping -h 127.0.0.1 --silent`
  (answers without credentials — no password on a command line). User and database come from
  `MYSQL_DATABASE: ${DB_DATABASE:?…}`, `MYSQL_USER: ${DB_USERNAME:?…}`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD` —
  **applied only on an empty volume**: changing them in `.env` later changes nothing in MySQL (change the user with SQL).
  `command:` flags:
  - `--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci`;
  - FULLTEXT ngram search: `--innodb-ft-enable-stopword=OFF --ngram-token-size=2` — InnoDB's stopword list (`a`, `i`,
    `in`, …) would drop every 2-character token containing them, and names like "Ali" become unsearchable;
  - memory for a 2 GB server: `--performance-schema=OFF --innodb-buffer-pool-size=256M --max-connections=40
    --temptable-max-ram=64M --innodb-log-buffer-size=8M --table-open-cache=400 --table-definition-cache=400
    --skip-name-resolve`. An idle MySQL takes ~145 MB instead of ~430 MB; performance_schema alone is ~250 MB of
    monitoring nobody reads. Raise the buffer pool only when the working set outgrows it and the server has the RAM.
- **`backup`** — `image: {{project}}-backup`, volume `backups:/backups`, `DB_*` from `.env` (the app user, not root),
  `BACKUP_KEEP_DAYS: ${BACKUP_KEEP_DAYS:-30}`, depends on healthy `db`.
- **All services:** `restart: unless-stopped`; logging `json-file` with `max-size: 10m`, `max-file: 3` (a YAML anchor
  `&logging` on the first service, `*logging` on the rest). *Why:* default Docker logs grow until the disk is full.
- **Volumes:** `storage` (uploaded files, `storage/app/private`), `db`, `backups`. Everything else in a container is
  disposable and is rebuilt every deploy.

## nginx inside the `web` container (`docker/nginx.conf`)

- **Real client IP and scheme only from the server's nginx:** `set_real_ip_from` for `10.0.0.0/8`, `172.16.0.0/12`,
  `192.168.0.0/16`, `127.0.0.1`; `real_ip_header X-Real-IP`; `map $http_x_forwarded_proto $forwarded_https { https on; }`
  → `fastcgi_param HTTPS $forwarded_https if_not_empty`. *Why:* without it every client is the Docker gateway (one IP
  for all rate limits) and Laravel thinks the request is HTTP (wrong URLs, no secure cookies).
- **Re-resolve the app container:** `resolver 127.0.0.11 valid=10s ipv6=off;` + `set $php_fpm app:9000;` +
  `fastcgi_pass $php_fpm;`. *Why:* a recreated `app` gets a new IP; a name resolved once at nginx start keeps pointing at
  the old one → 502 after every deploy.
- **Security headers:** CSP (`default-src 'self'`, `frame-ancestors 'none'`, …), `X-Frame-Options DENY`,
  `X-Content-Type-Options nosniff`, `Referrer-Policy strict-origin-when-cross-origin`, `Permissions-Policy`, all with
  `always`, `server_tokens off`. A `location` with its own `add_header` repeats them. *Why:* nginx does not inherit
  `add_header` into a location that defines any of its own — one cache header would silently drop the whole CSP there.
- **Rate limits** (`limit_req_status 429`), one zone per entry point (mini-crm: `crm_login` 10r/m on login,
  `crm_csrf` 30r/m on `/sanctum/`, `crm_api` 20r/s on `/api/` and `/files/`). *Why:* brute force and scraping are cut before PHP; the app adds its own per-account limit.
- **Body size:** `client_max_body_size 1M` by default; a larger limit (210M) only in a `location` matching the exact
  upload routes (in mini-crm: `^/api/((projects|clients)/[0-9]+/entries|projects/[0-9]+/payments|payments/[0-9]+)/?$`).
  *Why:* every other endpoint takes JSON; a 200 MB body on any URL would be a free way to fill PHP's temp disk.
- **Locations:** `/build/` — `Cache-Control: public, max-age=31536000, immutable` (Vite names carry a hash);
  `/files/` — private files, served by PHP (auth checked), never by nginx from disk; only `index.php` is executed
  (`internal`), any other `*.php` → 404; hidden files (`/.` except `.well-known`) — `deny all`.
- **`fastcgi_buffer_size 32k; fastcgi_buffers 16 16k;`** — Laravel's responses carry a long `Link` header (preloaded
  assets), more than the default 4–8 KB buffer → 502 "upstream sent too big header".

## PHP (`docker/php.ini`, `docker/php-fpm.conf`)

- `php.ini`: `upload_max_filesize 20M`, `post_max_size 210M`, `max_file_uploads 20` (must match the nginx upload limit),
  `memory_limit 256M`, `expose_php Off`, `date.timezone Asia/Dushanbe`.
- opcache: `opcache.enable=1`, `opcache.validate_timestamps=0`, `memory_consumption 128`, `max_accelerated_files 20000`;
  `realpath_cache_size 4096K`. *Why:* the code inside the image never changes — checking file times on every request is
  pure overhead. The flip side: code is never edited inside a running container; a change is a new deploy.
- `php-fpm.conf`: `listen = 9000`, `pm = dynamic`, `pm.max_children = 12` (`start 3`, `min_spare 2`, `max_spare 6`),
  `pm.max_requests = 1000`, error log to stderr, access log off.
  *Why:* `max_children` (12) stays well below MySQL `max-connections` (40): the `backup`, a `tinker` session and a
  migration still get a connection. Size `max_children` from RAM (~30–60 MB per worker), never above the DB connection
  limit.

## nginx on the server (`deploy/{{project}}.conf`)

- **HTTP only**, one `server` block per domain (mini-crm: `minicrm.hamd365.online`, `minicrm.hamd.tj`). Every block
  proxies to `http://127.0.0.1:{{PORT}}` (the `PORT` from `.env`), sets `Host`, `X-Real-IP`, `X-Forwarded-For`,
  `X-Forwarded-Proto`, `client_max_body_size 210m` (the web container narrows it to 1M per route),
  `proxy_buffer_size 32k` (the same long `Link` header). HTTPS, the redirect and HSTS are added by certbot (rule 3).
- **No `www`**: there is no DNS record for it, so no block and no certificate.
- **Sanctum on any domain:** `SANCTUM_STATEFUL_DOMAINS` is not set; `config/sanctum.php` includes
  `Sanctum::currentRequestHost()` in `stateful`. *Why:* the SPA is served from every domain of the server's nginx, and a
  page only calls the API of its own domain; a hard-coded list breaks login on every new domain.

## `deploy/deploy.sh`

`set -euo pipefail`, `cd` to the repo root, then in this order:

1. `git fetch origin && git reset --hard origin/main` — the server never has local changes; what is deployed is exactly `main`.
2. **Check `.env`, refuse to deploy otherwise:** the file exists; `APP_ENV=production`, `APP_DEBUG=false`; `APP_KEY` and
   `APP_URL` set; `SESSION_SECURE_COOKIE=true`; `DB_HOST=db`; `DB_USERNAME` is not `root`.
   *Why:* one wrong line here is a stack trace with secrets on the screen, cookies over HTTP, or the app as DB root —
   cheap to check, expensive to notice later.
3. `docker compose build --pull` and `docker compose pull --ignore-buildable` — fresh base images every deploy, so
   security releases of php/nginx/mysql arrive without anyone tracking them.
4. `docker compose run --rm app php artisan migrate --force` — `run` starts `db` itself and waits for it to be healthy.
   *Why this order:* the new images are built next to the running containers; if the build or a migration fails, the
   script stops and the site keeps running on the old containers.
5. `docker compose up -d --wait --remove-orphans` — `--wait` returns only when the healthchecks pass (a broken release
   makes the deploy fail loudly); `--remove-orphans` drops services deleted from `compose.yaml`.
6. `docker image prune -f --filter "label=project={{project}}"` — only this project's unnamed images.
7. `echo "Deployed $(git rev-parse --short HEAD)"`.

Generated docs (`php artisan swagger`) for API projects belong in the image build (the `app` stage, before `web` copies
`public/`), not in a CI step. **[proposal]** *Why:* the `web` image serves only what was in `public/` at build time.

## Backups

- The `backup` service sleeps until **02:00 Asia/Dushanbe (21:00 UTC)**, then `mysqldump --single-transaction --quick
  --routines --no-tablespaces | gzip` into `/backups/<db>-<UTC date>.sql.gz`.
- Written to `<file>.part` first, then `mv`: a half-written dump never looks finished; stale `.part` files are removed.
- The password goes through an option file (`mktemp`, `chmod 600`, `--defaults-extra-file`, deleted on exit), never as
  `-p<password>` — a command-line argument is visible to every user in `ps`.
- Dumps older than `BACKUP_KEEP_DAYS` (30) are deleted. A failed dump is logged and retried next night; the loop keeps running.
- Manual dump: `docker compose exec backup backup now`; list: `docker compose exec backup ls -lh /backups`.
- **Uploaded files are not in the dump.** They live in the `storage` volume; copy them with
  `docker compose cp app:/var/www/html/storage/app/private ./attachments-backup`.
- Restore (replaces the data): stop `web app`, `gunzip -c` the dump from the `backup` container into `mysql` in the `db`
  container, `up -d --wait`, then rebuild derived data if the project has such a command (mini-crm: `crm:rebuild-totals`).
- Backups on the same disk survive a bad migration, not a dead disk: copy them to another machine regularly.

## First launch

1. Clone into `/var/www/{{project}}`; `cp .env.example .env` and fill it: production values, passwords from
   `openssl rand -hex 24`, `PORT` — a free port on 127.0.0.1.
2. Server nginx + certbot (rule 3).
3. `deploy/deploy.sh`.
4. Once: `docker compose exec app php artisan migrate:fresh --seed --force` — the seeder creates the first admin
   (mini-crm: `admina`/`admina`) and demo data; **change the password immediately**. Or, without demo data, create the
   user with the project's command (mini-crm: `crm:user-create "Name" login`).
   After this, `migrate:fresh` is never run again (rule 6).

`.env.example` documents every production value next to its dev default (`# Production: APP_DEBUG=false`).

---

## Never do

| ❌ Forbidden | Why |
|---|---|
| `image: app:latest`, `:v1.2`, `:<sha>`, pushing images to a registry | The commit is the version; tags drift from the code [confirmed] |
| A stage without `LABEL project=…` | `docker image prune` cannot find it, old builds pile up on the disk |
| GitHub Actions / webhooks / auto-deploy on push; an assistant running `git push` or `deploy.sh` | No CI/CD: deploy is manual and watched [confirmed] |
| A symlink from `sites-available` to the repo, or HTTPS written into `deploy/*.conf` by hand | certbot owns HTTPS; a deploy would wipe it [confirmed] |
| One `server` block / one certificate for several domains | Domains must be independent |
| `.env` copied into the image, `ARG`/`ENV` with secrets, `DB_ROOT_PASSWORD` in `app` | Secrets leak with the image; the app never needs root [confirmed] |
| `php artisan optimize`/`config:cache` in the Dockerfile | Caches the build-time (empty) `.env` |
| Editing an applied migration; `migrate:fresh` or `migrate:rollback` in a deploy | Production schema drifts or data is lost [confirmed] |
| `ports: "8083:80"` (all interfaces) | Bypasses HTTPS and the real-IP logic of the server's nginx |
| `fastcgi_pass app:9000;` without `resolver` + variable | 502 after every recreated `app` container |
| `client_max_body_size 200M` for the whole site | Any URL becomes an upload endpoint |
| A `location` with its own `add_header` without the security headers | nginx drops the inherited CSP/frame headers there |
| `mysqldump -p"$PASSWORD"`, or `mysqldump` from a Debian/MariaDB image | Password visible in `ps`; MySQL rejects a MariaDB dump |
| `opcache.validate_timestamps=1`, editing code inside a running container | The image is immutable; a change is a deploy |
| `pm.max_children` ≥ MySQL `max-connections` | Backup, tinker and migrations get "Too many connections" |
| Docker logs without `max-size`/`max-file` | The disk fills up |
| `public/hot` in the build context | Production loads its assets from the Vite dev server |

## Checklist for a deploy-related change

- [ ] Every Dockerfile stage has `LABEL project={{project}}` (directly or via `FROM php`); `compose.yaml` images have no tags.
- [ ] `.dockerignore` still excludes `.env*`, `public/hot`, `vendor`, `node_modules`, `storage/*` contents, `bootstrap/cache/*.php`.
- [ ] A new secret or setting is added to `.env.example` (with its production value in a comment) and, if it must hold,
      to the `.env` check in `deploy.sh`.
- [ ] A new upload route: its regex in the 210M `location`, and `upload_max_filesize`/`post_max_size` agree with it.
- [ ] A new `location` with `add_header` repeats the security headers.
- [ ] Schema change = a new migration; old ones untouched.
- [ ] `deploy/README.md` (the human's runbook) updated if a step changed; a changed `deploy/{{project}}.conf` is
      reported to the author: it needs a fresh copy on the server + `certbot --reinstall` for every domain.
