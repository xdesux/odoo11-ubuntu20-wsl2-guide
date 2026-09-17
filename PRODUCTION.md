# Production deployment of the legacy Odoo 11 instance

The root [configuration](myodoo11.conf) and [service](myodoo11.service) are hardened templates for one database, local PostgreSQL, and a trusted HTTPS reverse proxy on the same Linux host. This document replaces the README's development settings for deployment, permissions, logging, and updates.

These files do not certify a server as production-ready. Python 3.6 is end-of-life; establish a supported upgrade path or a documented legacy patching arrangement before handling production data. Review Odoo and dependency support with your vendor. Prefer a dedicated Linux server/VM with a supported OS and tested application dependencies. Do not assume changing the OS alone makes Odoo 11 compatible or supported.

WSL2 is useful for testing these files. If retained for production, explicitly test Windows reboot, WSL startup, sleep, networking, backups, and recovery. systemd must be PID 1; enabling a unit alone does not establish Windows/WSL availability.

## Assumptions and sizing

- Linux account, PostgreSQL role and database: `myodoo11`.
- Existing source: `/opt/myodoo11/myodoo11`; virtualenv: `/opt/myodoo11/myodoo11-venv`.
- Python executable and wkhtmltopdf are already installed and tested.
- Only reviewed production addons are loaded. Create `custom_apps` if absent; do not load `test_apps`.
- Start with 2 vCPU and at least 4 GiB RAM for a small workload, then measure. Two HTTP workers, one cron worker and a longpolling process require memory in addition to PostgreSQL, the OS, and PDF rendering.
- Memory values are per-worker limits, not a total service memory cap. HTTP requests have a 60-second CPU/120-second wall-clock budget; cron has a 300-second wall-clock budget. Tune with real reports and imports.
- `db_maxconn=16` is per process. Budget PostgreSQL connections across all Odoo processes, maintenance clients, and other applications.

If changing the project/database name, update BOTH templates, all paths, the service preflight, and the exact database regex. Escape regex characters if needed.

## 1. Prepare and back up

Complete the source, virtualenv, PostgreSQL and PDF-tool installation first. Run the following from a checkout of this guide on the Linux host, as an administrator.

For an existing instance, enter a maintenance window and stop Odoo before changing anything:

```bash
sudo systemctl stop myodoo11
```

Back up the database, filestore, installed configuration and service before proceeding. Keep the old code/virtualenv revision available. Do not overwrite an existing config with the template until its site-specific values have been recorded.

## 2. Restrict database access

Use a local Unix socket and peer authentication: the OS user authenticates as the identically named PostgreSQL role, without a password in the config.

For a NEW installation only:

```bash
sudo -u postgres createuser --no-superuser --no-createdb --no-createrole myodoo11
sudo -u postgres createdb --owner=myodoo11 --encoding=UTF8 --template=template0 myodoo11
```

For an existing database, retain its contents and ownership. Adapt the templates if its name differs. Do not create or initialize a replacement database.

Open the cluster's actual `pg_hba.conf` (locate it with `sudo -u postgres psql -Atc 'SHOW hba_file'`). Add these entries BEFORE broader matching local entries:

```text
local   myodoo11   myodoo11   peer
local   all        myodoo11   reject
```

Keep administrator access intact. Reload PostgreSQL and verify the entries for your cluster. Remove unneeded TCP access for this role and keep PostgreSQL off public interfaces; the socket-only application config does not secure other PostgreSQL listeners.

Revoke the development role privileges and remove any old password:

```bash
sudo -u postgres psql -v ON_ERROR_STOP=1 -c "ALTER ROLE myodoo11 NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION NOBYPASSRLS PASSWORD NULL;"
sudo -u postgres psql -c "SELECT pg_reload_conf();"
sudo -u myodoo11 psql -X -w -h /var/run/postgresql -p 5432 -d myodoo11 -c 'SELECT current_user, current_database();'
```

The application role should own only this application's database and required objects. Review existing grants and role memberships; the ALTER ROLE command does not remove inherited access. Administrators create/restore databases; the web database manager stays disabled.

## 3. Install configuration and persistent state

```bash
sudo install -d -o myodoo11 -g myodoo11 -m 0700 /var/lib/myodoo11
sudo install -d -o root -g myodoo11 -m 0750 /opt/myodoo11/myodoo11/custom_apps
sudo install -o root -g myodoo11 -m 0640 myodoo11.conf /etc/myodoo11.conf
sudoedit /etc/myodoo11.conf
```

Generate a unique secret locally with `openssl rand -hex 32`. Paste the resulting 64 hexadecimal characters into the installed config as `admin_passwd = YOUR_SECRET`. Keep the exact spacing. The service deliberately refuses to start with the placeholder, a missing key, or a secret that does not match the 64-alphanumeric-character format. Do not commit the installed config or paste the secret into issue trackers.

For EXISTING installations, migrate the old data directory before starting. Locate the previous effective `data_dir` (often `/opt/myodoo11/.local/share/Odoo`), then copy its contents to `/var/lib/myodoo11` while Odoo is stopped. Preserve `filestore/<database>` and sessions, set ownership to `myodoo11:myodoo11`, and restrict directory/file access. Keep a backup of the original. A database restore without its matching filestore can lose attachments.

Make application code and the virtualenv read-only to the service user:

```bash
sudo chown -R root:myodoo11 /opt/myodoo11
sudo chmod -R u+rwX,g+rX,g-w,o-rwx /opt/myodoo11
```

Also ensure the Python installation under `/opt/python3.6` and PDF binaries are administrator-owned and executable by the service account. Code updates must now be performed by a trusted deployment administrator.

For a NEW, empty database only, initialize before starting the service:

```bash
sudo -u myodoo11 env PYTHONDONTWRITEBYTECODE=1 /opt/myodoo11/myodoo11-venv/bin/python /opt/myodoo11/myodoo11/odoo-bin -c /etc/myodoo11.conf -d myodoo11 -i base --without-demo=all --no-http --workers=0 --stop-after-init
```

Change the initial application administrator password in a restricted local session before allowing user access. The application login password and `admin_passwd` are separate secrets.

## 4. Install and start the service

```bash
sudo install -o root -g root -m 0644 myodoo11.service /etc/systemd/system/myodoo11.service
sudo systemd-analyze verify /etc/systemd/system/myodoo11.service
sudo systemctl daemon-reload
sudo systemctl enable --now myodoo11
sudo systemctl status myodoo11 --no-pager
sudo journalctl -u myodoo11 -n 100 --no-pager
```

Use `systemctl restart myodoo11` when applying a changed config to an already running service. If restart attempts were exhausted, fix the cause, then run `systemctl reset-failed myodoo11` and start it again.

The preflight checks actual database connectivity. PostgreSQL is wanted rather than coupled with Requires, so its restart does not permanently stop Odoo. Verify application recovery after an actual database restart in staging.

The service has a read-only filesystem except its state directory and private temporary storage. Custom addons requiring other writable paths need a narrowly scoped override after review. Test PDF rendering, attachments, email and integrations with the sandbox enabled. Logs go to journald, replacing the old log-file paths. Configure persistent journal storage and bounded disk usage/retention according to the host's policy; collect logs centrally and alert on errors/restarts.

## 5. HTTPS reverse proxy is required

Bind Nginx to your intended public/private interface and install a valid certificate for your actual domain. Serve only the expected hostname; reject unmatched hosts in a default server. Expose only HTTPS (and HTTP for redirect/certificate validation) to intended clients. Do not expose 5511, 5512 or PostgreSQL through firewall rules or WSL port forwarding.

Within the HTTPS virtual host, use the following routing. Replace the hostname and certificate paths in your site's server block. This is a fragment, not a complete TLS configuration:

```nginx
client_max_body_size 25m;
proxy_connect_timeout 10s;
proxy_send_timeout 130s;
proxy_read_timeout 130s;

# Overwrite client-supplied forwarding headers at this trusted edge.
proxy_set_header Host $host;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-For $remote_addr;
proxy_set_header X-Real-IP $remote_addr;

location ^~ /web/database {
    return 404;
}
location /longpolling {
    proxy_pass http://127.0.0.1:5512;
}
location / {
    proxy_redirect off;
    proxy_pass http://127.0.0.1:5511;
}
```

Use an HTTP-to-HTTPS redirect and automatic certificate renewal; run `nginx -t` before reloading. If there is another trusted load balancer, explicitly configure its trusted source addresses and real-IP processing; do not trust arbitrary incoming forwarded headers.

The Odoo config enables `proxy_mode` only for this topology. Direct HTTP access is a local diagnostic path, not a production login URL. Set Odoo's `web.base.url` to the real HTTPS URL and `web.base.url.freeze` to True. Verify PDF asset loading and any `report.url` override against your hostname/database filter.

## 6. Deployment acceptance and recovery

Before opening access:

- Confirm `ss -lntp` shows Odoo only on 127.0.0.1:5511 and :5512; verify the ports are inaccessible from another machine.
- Check `curl -I http://127.0.0.1:5511/web/login` locally, then log in through HTTPS. Redirects may be normal; a status code alone does not prove application health.
- Confirm database management routes are blocked and only the intended database is served.
- Test attachments, PDF reports with assets, scheduled jobs, longpolling/chat, outgoing mail and relevant custom workflows.
- Load-test expected concurrent users and imports. Watch worker recycling, memory, database connections, response times and disk growth before tuning limits.
- Test service and database restart recovery, host reboot and, if applicable, WSL shutdown/startup.
- Schedule encrypted off-host backups of the database AND matching filestore plus configuration; perform and time a full restore into an isolated environment.
- Monitor HTTPS availability, certificate expiry, failed jobs, backup freshness, service failures and storage.

For upgrades, stop the service, take a consistent database/filestore backup, deploy a pinned reviewed revision as administrator, and run required module migrations as `myodoo11` with `--no-http --workers=0 --stop-after-init`. Restart and repeat acceptance checks. Do not run `git pull` as the runtime user. If a migration fails, rollback may require restoring the database and filestore along with the previous code and config.

## Validation and references

These templates were reviewed against Odoo 11 configuration/server source. They have not been run against your application, database, Ubuntu systemd, reverse proxy or workload. Complete the host checks above before production use.

- [Odoo 11 configuration options](https://github.com/odoo/odoo/blob/11.0/odoo/tools/config.py)
- [Odoo 11 server implementation](https://github.com/odoo/odoo/blob/11.0/odoo/service/server.py)
- [Odoo deployment guidance](https://www.odoo.com/documentation/14.0/administration/install/deploy.html) (later-version general guidance; use the Odoo 11 source for option compatibility)
- [Python version lifecycle](https://devguide.python.org/versions/)
