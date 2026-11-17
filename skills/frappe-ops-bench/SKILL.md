---
name: "frappe-ops-bench"
description: "Use when running bench commands on the ERPNext v16 server: sites, apps, migrate, build, backup/restore, scheduler, config, console, production setup and troubleshooting."
---

# Bench CLI Reference (Frappe v14–v16)

Source: OpenAEC-Foundation/Frappe_Claude_Skill_Package (LGPL-3.0), adapted.

In a Docker (frappe_docker) setup run bench inside the backend container: `docker compose exec backend bench --site <site> <cmd>`. Read/inspect freely there, but never create or edit apps inside the container (use apps.json + GitHub + rebuild).

## Essential Commands

| Task | Command |
|---|---|
| Create bench | `bench init proj --frappe-branch version-16` |
| Create site | `bench new-site site.local --admin-password X` |
| Default site | `bench use site.local` |
| Get app | `bench get-app erpnext --branch version-16` |
| Install app | `bench --site S install-app erpnext` |
| List apps | `bench --site S list-apps` |
| Migrate | `bench --site S migrate` |
| Build assets | `bench build --app myapp` |
| Backup | `bench --site S backup --with-files` |
| Restore | `bench --site S restore db.sql.gz --with-public-files f.tar --with-private-files p.tar` |
| Console | `bench --site S console` |
| DB shell | `bench --site S mariadb` |
| Run method | `bench --site S execute app.module.fn --kwargs '{"a":1}'` |
| Scheduler health | `bench doctor` / `bench show-pending-jobs` |
| Clear cache | `bench --site S clear-cache` |
| Drop site | `bench drop-site S --force` |

Run from inside the bench dir; never as root; always pin `--frappe-branch`.

## App management

```bash
bench get-app https://github.com/org/custom-app.git --branch main
bench --site S install-app custom_app
bench --site S uninstall-app custom_app   # backup first!
bench remove-app custom_app               # never delete folders manually
bench switch-to-branch version-16 frappe erpnext
```

## bench update

Order: backup → git pull → requirements → build → migrate → restart. Run in tmux/screen. NEVER `--reset` in prod without understanding it does `git reset --hard`. Test on staging first. (Docker: update = rebuild image + `migrate`.)

## bench migrate

Schema sync → patches → search index → translations → dashboard cache. Run after hooks/DocType/code changes. `bench --site all migrate` for all sites. Skipping causes schema mismatch.

## bench build

After JS/CSS or package.json changes: `bench build --app myapp` (or `--production`); dev: `bench watch`.

## Backup & restore

Backups go to `sites/S/private/backups/` (database .sql.gz, files .tar, private-files .tar). Default retention ~24h → set `keep_backups_for_hours`. ALWAYS backup before update, uninstall-app, drop-site; test restores on staging.

## Scheduler & jobs

```bash
bench --site S scheduler enable|disable|pause|resume
bench --site S purge-jobs
bench --site S trigger-scheduler-event hourly
bench worker --queue short
```
v16 scheduler tick is 60s.

## Multi-tenancy / domains

```bash
bench config dns_multitenant on
bench new-site erp.example.com --admin-password X
bench setup add-domain erp.example.com --site S
bench setup nginx && sudo service nginx reload
```

## common_site_config.json (all sites)

`background_workers`, `developer_mode`, `dns_multitenant`, `gunicorn_workers` (≥2), `maintenance_mode`, `pause_scheduler`, `serve_default_site`, `server_script_enabled`, `webserver_port` (8000), `socketio_port` (9000), `redis_cache`, `redis_queue`, `redis_socketio`.

```bash
bench config set-common-config -c server_script_enabled 1
bench --site S set-config developer_mode 1
bench --site S show-config
```

## site_config.json (per site)

Required: `db_type`, `db_name`, `db_password`. Useful: `host_name`, `allow_cors`, `max_file_size`, `mute_emails`, `logging` (2 = show SQL). v16 supports separate `db_user`.
Priority: env var (`FRAPPE_DB_HOST`, `FRAPPE_REDIS_QUEUE`, …) > site_config > common_site_config > default.

## Production (non-Docker)

```bash
sudo bench setup production frappe-user
bench setup lets-encrypt erp.example.com
sudo bench restart
```

## Troubleshooting quick list

- Changes not showing → clear-cache, build, restart
- "DocType not found" after pull → migrate
- Jobs stuck → `bench doctor`, check workers/redis, `show-pending-jobs`
- Server Script not running → `server_script_enabled` = 1
- 502 → check gunicorn/supervisor (or container) logs in `logs/`