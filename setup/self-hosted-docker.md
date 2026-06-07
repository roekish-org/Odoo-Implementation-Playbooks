# Self-Hosted Odoo on Docker — Dev → Staging → Production

> A reproducible, Dockerised Odoo stack you can stand up in minutes and grow into a real
> staging/production deployment. Pinned **Odoo 19.0** (current stable) + **PostgreSQL 16**.
> This is the exact shape of the stack Roekish uses internally — published so you can run it yourself.

**Region note (France / EU):** for production, host in an EU datacenter (or use Odoo.sh EU region)
to keep customer data inside the EU. GDPR notes are flagged inline as **🇪🇺**.

| | |
|---|---|
| Odoo version | 19.0 (current stable; works on 18.0 / 17.0 — change the image tag) |
| Database | PostgreSQL 16 |
| Effort (dev stack) | ~30 minutes |
| Effort (hardened staging) | ~half a day |
| Prerequisites | Docker + Docker Compose v2, a terminal |

---

## 1. Project layout

```
odoo-stack/
├── docker-compose.yml
├── .env.example          # committed — documents every tunable
├── .env                  # NOT committed — your real values
├── config/
│   └── odoo.conf         # committed — contains NO secrets
└── addons/               # your custom modules (bind-mounted)
```

The golden rule that makes this safe to put in git: **no secret ever lives in a committed
file.** `odoo.conf` carries no DB password (the entrypoint injects it from env), and `.env`
is git-ignored.

`.gitignore`:

```gitignore
.env
addons/**/*.pyc
__pycache__/
```

---

## 2. The compose file

```yaml
# docker-compose.yml — Odoo 19 + PostgreSQL 16
services:
  db:
    image: postgres:16
    container_name: ${COMPOSE_PROJECT_NAME:-odoo-dev}-db
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-odoo}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-odoo}
      POSTGRES_DB: postgres            # Odoo creates its own app databases
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-odoo} -d postgres"]
      interval: 5s
      timeout: 5s
      retries: 12
    restart: unless-stopped

  odoo:
    image: odoo:19.0          # current stable; pin to 18.0 / 17.0 if your project targets those
    container_name: ${COMPOSE_PROJECT_NAME:-odoo-dev}-odoo
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "${ODOO_PORT:-8069}:8069"          # web / JSON-RPC / XML-RPC
      - "${ODOO_GEVENT_PORT:-8072}:8072"   # websocket / longpolling
    environment:
      HOST: db
      PORT: 5432
      USER: ${POSTGRES_USER:-odoo}
      PASSWORD: ${POSTGRES_PASSWORD:-odoo}
    volumes:
      - odoo-web-data:/var/lib/odoo
      - ./config/odoo.conf:/etc/odoo/odoo.conf:ro
      - ./addons:/mnt/extra-addons
    restart: unless-stopped

volumes:
  db-data:
  odoo-web-data:
```

Two things worth understanding:

- **Why the DB user is a superuser.** The official `postgres` image makes `POSTGRES_USER` a
  superuser, which Odoo needs in order to `CREATE DATABASE` on demand from the database manager.
- **Why DB credentials are env, not config.** The Odoo entrypoint reads `HOST`/`PORT`/`USER`/`PASSWORD`
  and builds the connection only when `odoo.conf` does **not** set `db_host`/`db_user`/etc. Leaving
  them out of the committed config is exactly what keeps secrets out of git.

---

## 3. Configuration (`config/odoo.conf`)

```ini
[options]
; DB connection is intentionally NOT set here — the entrypoint injects it from
; HOST/USER/PASSWORD env vars. Keeping credentials out of this file lets us commit it.

addons_path = /mnt/extra-addons
data_dir    = /var/lib/odoo

; Master password for /web/database/manager. DEV-ONLY default — override everywhere else.
admin_passwd = admin

; Database manager visible so engineers can create/drop dev DBs from the browser.
list_db = True

log_level = info
logfile   = False

; Single process = predictable dev/debugging. Raise workers for staging/load.
workers          = 0
max_cron_threads = 1
```

`.env.example` (copy to `.env` and edit):

```dotenv
COMPOSE_PROJECT_NAME=odoo-dev
POSTGRES_USER=odoo
POSTGRES_PASSWORD=odoo          # change for anything shared
ODOO_PORT=8069
ODOO_GEVENT_PORT=8072
```

---

## 4. Bring it up

```bash
cp .env.example .env
docker compose up -d
docker compose logs -f odoo          # wait for "HTTP service (werkzeug) running"
```

Open <http://localhost:8069>, create a database (master password = `admin_passwd`), and you have
a working Odoo. To pre-install modules without the wizard:

```bash
docker compose run --rm odoo odoo -d demo -i base,l10n_fr --stop-after-init
```

### Useful one-liners

| Goal | Command |
|---|---|
| Install a module | `docker compose run --rm odoo odoo -d <db> -i <module> --stop-after-init` |
| Upgrade a module | `docker compose run --rm odoo odoo -d <db> -u <module> --stop-after-init` |
| Open a shell (ORM) | `docker compose run --rm odoo odoo shell -d <db>` |
| Tail logs | `docker compose logs -f odoo` |
| Drop everything | `docker compose down -v` (⚠️ deletes the DB volume) |

After editing a custom addon, restart Odoo and run an `-u <module>` upgrade so schema/view changes
are applied:

```bash
docker compose restart odoo
docker compose run --rm odoo odoo -d <db> -u <module> --stop-after-init
```

---

## 5. From dev to staging/production

The dev stack above is deliberately permissive. Before any shared environment, harden it:

| Setting | Dev | Staging / Production |
|---|---|---|
| `admin_passwd` | `admin` | long random secret, via env (`ADMIN_PASSWD`), **never** in git |
| `POSTGRES_PASSWORD` | `odoo` | long random secret |
| `list_db` | `True` | **`False`** — hide the database manager |
| `workers` | `0` | `(2 × CPU cores) + 1`, set `limit_*` memory caps |
| `proxy_mode` | unset | **`True`** (Odoo behind a reverse proxy) |
| TLS | none | terminate HTTPS at a reverse proxy (Caddy / nginx / Traefik) |
| Backups | none | automated `pg_dump` + filestore snapshot, off-box, tested restore |
| Port `8069` | published | **not** published publicly — proxy only |

### Reverse proxy + TLS (Caddy example)

Caddy gives you automatic Let's Encrypt certificates with almost no config:

```caddy
odoo.example.com {
    reverse_proxy odoo:8069
    # Odoo's websocket / longpolling worker:
    @ws path /websocket /longpolling/*
    reverse_proxy @ws odoo:8072
}
```

With a proxy in front, set `proxy_mode = True` in `odoo.conf` and stop publishing `8069`/`8072`
to the host — expose only Caddy's `80`/`443`.

### Production worker sizing

```ini
workers              = 5          ; (2 × cores) + 1 for a 2-core box
max_cron_threads     = 2
limit_memory_soft    = 2147483648 ; 2 GB
limit_memory_hard    = 2684354560 ; 2.5 GB
limit_time_cpu       = 600
limit_time_real      = 1200
```

---

## 6. Backups (do this before go-live, not after)

A backup you have never restored is not a backup. Minimum viable, tested daily:

```bash
# Database
docker compose exec -T db pg_dump -U "$POSTGRES_USER" -Fc <db> > "backup-$(date +%F).dump"

# Filestore (attachments, sessions) — lives in the odoo-web-data volume
docker run --rm -v odoo-stack_odoo-web-data:/data -v "$PWD":/out alpine \
  tar czf /out/filestore-$(date +%F).tgz -C /data .
```

**🇪🇺 GDPR:** store backups inside the EU, encrypt them at rest, set a retention window, and make
sure your restore procedure can also honour deletion/erasure requests.

Restore drill (run it monthly):

```bash
docker compose exec -T db createdb -U "$POSTGRES_USER" <db>_restore
docker compose exec -T db pg_restore -U "$POSTGRES_USER" -d <db>_restore < backup-YYYY-MM-DD.dump
```

---

## 7. Upgrading Odoo versions

1. Snapshot DB + filestore (section 6) and **test the restore**.
2. Bump the image tag (e.g. `odoo:18.0` → `odoo:19.0`) on a **copy** of production data.
3. Run the standard upgrade against the copy; review the upgrade log for deprecations.
4. Re-run UAT (see [`../templates/uat-sign-off.md`](../templates/uat-sign-off.md)) before promoting.

Never upgrade in place without a tested rollback path.

---

## Validation checklist

- [ ] `docker compose up -d` brings up `db` (healthy) and `odoo`.
- [ ] <http://localhost:8069> loads and you can create a database.
- [ ] A custom module in `addons/` is installable with `-i`.
- [ ] No secret is present in any committed file (`git grep -iE 'passwd|password' -- '*.conf' '*.yml'` shows only env placeholders).
- [ ] **Before staging/prod:** `list_db = False`, real `admin_passwd`/DB password, TLS via proxy, `proxy_mode = True`, port `8069` not publicly exposed.
- [ ] A database backup has been taken **and restored** at least once.

---

*Part of the [Roekish Odoo Implementation Playbooks](../README.md). Honest, reproducible, EU-aware.*
