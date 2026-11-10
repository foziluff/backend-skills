# Deploy templates

Production Docker for a Laravel project, copied from `mini-crm` as it runs in production.
Rules and reasons: [references/deployment.md](../../references/deployment.md).
The comments inside are *why* comments of the real files — keep them; delete only the `# DECISION:` notes once decided.

## Placeholders

| Placeholder | Example (mini-crm) |
|---|---|
| `{{project}}` | `mini-crm` — compose `name`, image names, `LABEL project=…`, volume prefix, conf file name |
| `{{Project Name}}` | `Mini CRM` |
| `{{PORT}}` | `8083` — a free port on 127.0.0.1, the same as `PORT` in `.env` |
| `{{domain}}` | `minicrm.hamd365.online` — one `server` block per domain |
| `{{upload_routes_regex}}` | `^/api/((projects\|clients)/[0-9]+/entries\|projects/[0-9]+/payments\|payments/[0-9]+)/?$` |

## Where each file goes

| Template | Project path |
|---|---|
| [Dockerfile.stub](Dockerfile.stub) | `Dockerfile` |
| [dockerignore.stub](dockerignore.stub) | `.dockerignore` |
| [compose.yaml.stub](compose.yaml.stub) | `compose.yaml` |
| [php.ini.stub](php.ini.stub) | `docker/php.ini` |
| [php-fpm.conf.stub](php-fpm.conf.stub) | `docker/php-fpm.conf` |
| [entrypoint.sh.stub](entrypoint.sh.stub) | `docker/entrypoint.sh` |
| [nginx.conf.stub](nginx.conf.stub) | `docker/nginx.conf` |
| [nginx-php.conf.stub](nginx-php.conf.stub) | `docker/nginx-php.conf` |
| [backup.sh.stub](backup.sh.stub) | `docker/backup.sh` |
| [deploy.sh.stub](deploy.sh.stub) | `deploy/deploy.sh` (`chmod +x`) |
| [server-nginx.conf.stub](server-nginx.conf.stub) | `deploy/{{project}}.conf` — copied to the server, never linked |

Also needed in the project: `config/sanctum.php` with `Sanctum::currentRequestHost()` in `stateful`; `.env.example` with
the production values in comments (`PORT`, `BACKUP_KEEP_DAYS`, `DB_HOST=db`, `SESSION_SECURE_COOKIE=true`);
`deploy/README.md` — the runbook for a human (first launch, rollback, backups, restore) in the author's language.
