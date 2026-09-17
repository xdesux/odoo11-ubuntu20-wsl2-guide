# Odoo 11 Setup Guide — Ubuntu 20.04 WSL2

> **Deployment templates:** Use [PRODUCTION.md](PRODUCTION.md) with [myodoo11.conf](myodoo11.conf) and [myodoo11.service](myodoo11.service) for the hardened legacy setup. It supersedes the development examples below for configuration, database privileges, service installation, permissions, logging, network access and updates. The templates require local PostgreSQL peer authentication, an existing database, a locally generated master password, and a same-host HTTPS proxy. Complete the migration and validation steps before use.

## 1. Purpose

This document provides a reusable step-by-step process for installing and configuring **Odoo 11** on **Ubuntu 20.04 running under WSL2**.

This guide is intended for development, testing, staging, or legacy Odoo 11 environments.

The setup uses:

- Ubuntu 20.04 WSL2
- Odoo 11
- Python 3.6.15
- Python virtual environment
- PostgreSQL
- wkhtmltopdf
- systemd service
- Custom Odoo addons
- Dedicated Linux and PostgreSQL users

---

# 2. Installation Variables

Before starting, define the values for the project.

Replace the example values below with the actual project details.

| Variable | Example |
|---|---|
| Project Name | `myodoo11` |
| Linux User | `myodoo11` |
| Installation Directory | `/opt/myodoo11` |
| Odoo Source Directory | `/opt/myodoo11/myodoo11` |
| Virtual Environment | `/opt/myodoo11/myodoo11-venv` |
| Odoo Config | `/etc/myodoo11.conf` |
| Odoo Log Directory | `/var/log/myodoo11` |
| Odoo Log File | `/var/log/myodoo11/myodoo11-server.log` |
| Systemd Service | `myodoo11.service` |
| PostgreSQL User | `myodoo11` |
| PostgreSQL Password | `CHANGE_ME` |
| PostgreSQL Port | `5432` |
| Odoo Port | `5511` |
| Git Repository | `https://github.com/organization/repository.git` |

Throughout this guide, replace:

```text
myodoo11
```

with the actual project name.

---

# 3. Update Ubuntu

Run as your normal WSL user:

```bash
sudo apt update
sudo apt upgrade -y
```

Optional:

```bash
sudo apt-get dist-upgrade -y
```

---

# 4. Install Required System Packages

Install the required build tools and libraries:

```bash
sudo apt install -y \
    git \
    wget \
    curl \
    build-essential \
    gcc \
    g++ \
    make \
    software-properties-common \
    libssl-dev \
    zlib1g-dev \
    libncurses5-dev \
    libncursesw5-dev \
    libreadline-dev \
    libsqlite3-dev \
    libgdbm-dev \
    libdb5.3-dev \
    libbz2-dev \
    libexpat1-dev \
    liblzma-dev \
    tk-dev \
    libffi-dev \
    uuid-dev \
    libxml2-dev \
    libxslt1-dev \
    libevent-dev \
    libsasl2-dev \
    libldap2-dev \
    libpq-dev \
    libjpeg-dev \
    libpng-dev \
    libfreetype6-dev \
    libblas-dev \
    liblapack-dev \
    libopenblas-dev \
    liblcms2-dev \
    libwebp-dev \
    libharfbuzz-dev \
    libfribidi-dev \
    libxcb1-dev \
    node-less \
    npm
```

---

# 5. Create a Dedicated Odoo Linux User

Create a system user for the Odoo instance:

```bash
sudo useradd -m -U -r \
    -d /opt/myodoo11 \
    -s /bin/bash \
    myodoo11
```

Verify:

```bash
id myodoo11
```

Set ownership:

```bash
sudo chown -R myodoo11:myodoo11 /opt/myodoo11
```

Switch to the Odoo user:

```bash
sudo su - myodoo11
```

---

# 6. Clone the Odoo Repository

Clone the project's Odoo 11 source repository:

```bash
git clone https://github.com/organization/repository.git \
    /opt/myodoo11/myodoo11
```

For a private repository, GitHub authentication may be required.

Verify the repository:

```bash
ls -la /opt/myodoo11/myodoo11
```

Typical Odoo 11 repository contents:

```text
addons/
odoo/
odoo-bin
requirements.txt
setup.py
custom_apps/
test_apps/
```

The custom addon folders depend on the project.

---

# 7. Install Python 3.6.15

Ubuntu 20.04 uses Python 3.8 by default.

Odoo 11 legacy environments commonly use Python 3.6, so Python 3.6.15 should be installed separately without replacing Ubuntu's system Python.

Exit the Odoo user:

```bash
exit
```

Download Python:

```bash
cd /tmp
wget https://www.python.org/ftp/python/3.6.15/Python-3.6.15.tgz
```

Extract:

```bash
tar -xzf Python-3.6.15.tgz
cd Python-3.6.15
```

Configure:

```bash
./configure \
    --prefix=/opt/python3.6 \
    --enable-optimizations \
    --with-ensurepip=install
```

Compile:

```bash
make -j$(nproc)
```

Install:

```bash
sudo make altinstall
```

## Why use `altinstall`?

Use:

```bash
sudo make altinstall
```

instead of:

```bash
sudo make install
```

`altinstall` prevents Python 3.6 from replacing Ubuntu's default system Python.

Replacing Ubuntu's system Python can break operating system utilities.

Verify:

```bash
/opt/python3.6/bin/python3.6 --version
```

Expected:

```text
Python 3.6.15
```

---

# 8. Create the Python Virtual Environment

Switch back to the Odoo user:

```bash
sudo su - myodoo11
```

Go to the installation directory:

```bash
cd /opt/myodoo11
```

Create the virtual environment:

```bash
/opt/python3.6/bin/python3.6 -m venv myodoo11-venv
```

Activate:

```bash
source /opt/myodoo11/myodoo11-venv/bin/activate
```

Verify:

```bash
python --version
```

Expected:

```text
Python 3.6.15
```

---

# 9. Install Compatible pip and setuptools

Older Odoo 11 dependencies are not compatible with the latest Python packaging tools.

Install compatible versions:

```bash
python -m pip install --upgrade "pip<22"
```

Install a compatible setuptools version:

```bash
pip install setuptools==57.5.0
```

Install wheel:

```bash
pip install wheel
```

Verify:

```bash
pip --version
```

And:

```bash
python -c "import setuptools; print(setuptools.__version__)"
```

Expected setuptools:

```text
57.5.0
```

---

# 10. Install Odoo Python Requirements

Run:

```bash
pip install -r /opt/myodoo11/myodoo11/requirements.txt
```

If the following error appears:

```text
error in feedparser setup command:
use_2to3 is invalid
```

ensure setuptools is pinned:

```bash
pip install setuptools==57.5.0
```

Then retry:

```bash
pip install -r /opt/myodoo11/myodoo11/requirements.txt
```

---

# 11. Install Additional Python Packages

Common additional Odoo 11 packages:

```bash
pip install python-openid
```

If the project requires Google Data APIs:

```bash
pip install gdata
```

Optional:

```bash
pip install phonenumbers
```

The `phonenumbers` package removes warnings such as:

```text
The `phonenumbers` Python module is not available.
```

When finished:

```bash
deactivate
```

Exit the Odoo user:

```bash
exit
```

---

# 12. Install wkhtmltopdf

Odoo uses wkhtmltopdf for PDF report generation.

Check whether wkhtmltopdf is already installed:

```bash
which wkhtmltopdf
```

Verify:

```bash
wkhtmltopdf --version
```

Example location:

```text
/usr/local/bin/wkhtmltopdf
```

Odoo logs should eventually show:

```text
Will use the Wkhtmltopdf binary at /usr/local/bin/wkhtmltopdf
```

Also check:

```bash
which wkhtmltoimage
```

---

# 13. Install and Verify PostgreSQL

Install PostgreSQL if it is not already available:

```bash
sudo apt install postgresql postgresql-contrib -y
```

Check clusters:

```bash
pg_lsclusters
```

Example:

```text
Ver Cluster Port Status Owner    Data directory
12  main    5432 online postgres /var/lib/postgresql/12/main
```

The PostgreSQL port in the Odoo config must match the port shown here.

---

# 14. Create the PostgreSQL Odoo User

Open PostgreSQL:

```bash
sudo -u postgres psql
```

Create the Odoo role:

```sql
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1
        FROM pg_roles
        WHERE rolname = 'myodoo11'
    ) THEN
        CREATE ROLE myodoo11
        LOGIN
        PASSWORD 'CHANGE_ME';
    ELSE
        ALTER ROLE myodoo11
        WITH LOGIN
        PASSWORD 'CHANGE_ME';
    END IF;
END
$$;
```

Allow Odoo to create databases:

```sql
ALTER ROLE myodoo11 CREATEDB;
```

Check:

```sql
\du
```

Expected:

```text
Role name | Attributes
----------+-----------
myodoo11  | Create DB
```

Exit:

```sql
\q
```

For most Odoo environments, the Odoo database user does not need:

```text
SUPERUSER
CREATEROLE
```

---

# 15. Create the Odoo Log Directory

Create:

```bash
sudo mkdir -p /var/log/myodoo11
```

Set ownership:

```bash
sudo chown -R myodoo11:myodoo11 /var/log/myodoo11
```

Verify:

```bash
ls -ld /var/log/myodoo11
```

---

# 16. Create the Odoo Configuration File

Use the repository's [myodoo11.conf](myodoo11.conf). Follow [production setup steps 1–3](PRODUCTION.md) to prepare database access, permissions, the master password and persistent state before starting Odoo. Do not use the earlier development database credentials with this template.

---

# 17. Test Odoo Manually

Before creating the system service, always verify that Odoo starts manually.

Switch to the Odoo user:

```bash
sudo su - myodoo11
```

Activate the environment:

```bash
source /opt/myodoo11/myodoo11-venv/bin/activate
```

Go to the source directory:

```bash
cd /opt/myodoo11/myodoo11
```

Start Odoo:

```bash
python odoo-bin -c /etc/myodoo11.conf
```

You may see:

```text
Warn: Can't find .pfb for face 'Times-Roman'
```

This is usually a ReportLab font warning and does not necessarily prevent Odoo from starting.

Leave this terminal running.

---

# 18. Verify Odoo Port

Open another WSL terminal.

Check the configured port:

```bash
ss -lntp | grep 5511
```

Expected:

```text
LISTEN 0 128 0.0.0.0:5511 0.0.0.0:*
```

---

# 19. Test Odoo with curl

Run:

```bash
curl -I http://localhost:5511
```

Expected response:

```text
HTTP/1.0 200 OK
```

You may also see:

```text
Server: Werkzeug/... Python/3.6.15
```

This confirms that the Odoo HTTP server is working.

---

# 20. Check the Odoo Log

Run:

```bash
tail -n 100 /var/log/myodoo11/myodoo11-server.log
```

For live monitoring:

```bash
tail -f /var/log/myodoo11/myodoo11-server.log
```

Successful startup usually contains:

```text
Odoo version 11.0
```

and:

```text
HTTP service (werkzeug) running on ...
```

---

# 21. Stop the Manual Odoo Instance

Return to the first terminal and press:

```text
Ctrl + C
```

Deactivate the environment:

```bash
deactivate
```

Exit:

```bash
exit
```

---

# 22. Create the systemd Service

Use [myodoo11.service](myodoo11.service) and the [service installation procedure](PRODUCTION.md#4-install-and-start-the-service). It requires the matching configuration, local PostgreSQL peer authentication and the prepared database.

---

# 23. Reload systemd

Run:

```bash
sudo systemctl daemon-reload
```

Start Odoo:

```bash
sudo systemctl start myodoo11
```

Check:

```bash
sudo systemctl status myodoo11
```

Expected:

```text
Active: active (running)
```

---

# 24. Enable Automatic Startup

Run:

```bash
sudo systemctl enable myodoo11
```

Check:

```bash
sudo systemctl is-enabled myodoo11
```

Expected:

```text
enabled
```

---

# 25. Common Service Commands

Start:

```bash
sudo systemctl start myodoo11
```

Stop:

```bash
sudo systemctl stop myodoo11
```

Restart:

```bash
sudo systemctl restart myodoo11
```

Status:

```bash
sudo systemctl status myodoo11
```

View service logs:

```bash
sudo journalctl -u myodoo11
```

Live service logs:

```bash
sudo journalctl -u myodoo11 -f
```

---

# 26. Browser Access

From Windows, open:

```text
http://localhost:5511
```

Or:

```text
http://127.0.0.1:5511
```

---

# 27. Check Whether Odoo Is Listening

```bash
ss -lntp | grep 5511
```

Alternative:

```bash
sudo lsof -i :5511
```

---

# 28. Check PostgreSQL Connection

Test the Odoo database user:

```bash
psql \
    -h localhost \
    -p 5432 \
    -U myodoo11 \
    -d postgres
```

Enter the configured PostgreSQL password.

If successful:

```text
postgres=>
```

Exit:

```sql
\q
```

---

# 29. File Permission Recommendations

Do not use:

```bash
sudo chmod -R 777 /opt/myodoo11
```

Instead use correct ownership:

```bash
sudo chown -R myodoo11:myodoo11 /opt/myodoo11
```

Recommended permissions:

```bash
sudo chmod -R u+rwX,go+rX /opt/myodoo11
```

Check problematic files:

```bash
find /opt/myodoo11 -type f ! -perm -o+r -ls
```

---

# 30. Updating the Project Source

Switch to the Odoo user:

```bash
sudo su - myodoo11
```

Go to the repository:

```bash
cd /opt/myodoo11/myodoo11
```

Pull changes:

```bash
git pull
```

Exit:

```bash
exit
```

Restart Odoo:

```bash
sudo systemctl restart myodoo11
```

---

# 31. Updating a Custom Module

Example:

```bash
sudo su - myodoo11
source /opt/myodoo11/myodoo11-venv/bin/activate
cd /opt/myodoo11/myodoo11
```

Run:

```bash
python odoo-bin \
    -c /etc/myodoo11.conf \
    -d DATABASE_NAME \
    -u MODULE_NAME \
    --stop-after-init
```

Then:

```bash
deactivate
exit
```

Restart:

```bash
sudo systemctl restart myodoo11
```

---

# 32. Common Troubleshooting

## Odoo will not start

Check:

```bash
sudo systemctl status myodoo11
```

Then:

```bash
sudo journalctl -u myodoo11 -n 100
```

And:

```bash
tail -n 100 /var/log/myodoo11/myodoo11-server.log
```

---

## Port already in use

Check:

```bash
ss -lntp | grep 5511
```

or:

```bash
sudo lsof -i :5511
```

Either stop the existing application or change:

```ini
xmlrpc_port = 5511
```

to another available port.

---

## PostgreSQL connection refused

Check:

```bash
pg_lsclusters
```

Check service:

```bash
sudo systemctl status postgresql
```

Restart:

```bash
sudo systemctl restart postgresql
```

Confirm PostgreSQL port:

```bash
ss -lntp | grep 5432
```

---

## PostgreSQL password authentication failed

Reset password:

```bash
sudo -u postgres psql
```

Then:

```sql
ALTER ROLE myodoo11 WITH PASSWORD 'NEW_PASSWORD';
```

Exit:

```sql
\q
```

Make sure `/etc/myodoo11.conf` contains the same password.

---

## Python dependency error

Activate the environment:

```bash
sudo su - myodoo11
source /opt/myodoo11/myodoo11-venv/bin/activate
```

Check:

```bash
python --version
pip --version
```

Expected Python:

```text
Python 3.6.15
```

Check setuptools:

```bash
python -c "import setuptools; print(setuptools.__version__)"
```

Recommended:

```text
57.5.0
```

---

## Feedparser `use_2to3` error

Fix:

```bash
pip install setuptools==57.5.0
```

Then:

```bash
pip install -r /opt/myodoo11/myodoo11/requirements.txt
```

---

# 33. Useful Daily Commands

### Restart Odoo

```bash
sudo systemctl restart myodoo11
```

### Check status

```bash
sudo systemctl status myodoo11
```

### Watch Odoo logs

```bash
tail -f /var/log/myodoo11/myodoo11-server.log
```

### Watch systemd logs

```bash
sudo journalctl -u myodoo11 -f
```

### Check Odoo port

```bash
ss -lntp | grep 5511
```

### Check PostgreSQL

```bash
pg_lsclusters
```

### Check disk

```bash
df -h
```

### Check RAM

```bash
free -h
```

### Check Odoo process

```bash
ps aux | grep odoo
```

---

# 34. Recommended Project Layout

```text
/opt/myodoo11/
│
├── myodoo11/
│   ├── addons/
│   ├── custom_apps/
│   ├── test_apps/
│   ├── odoo/
│   ├── odoo-bin
│   ├── requirements.txt
│   └── ...
│
└── myodoo11-venv/
```

Configuration:

```text
/etc/myodoo11.conf
```

Service:

```text
/etc/systemd/system/myodoo11.service
```

Logs:

```text
/var/log/myodoo11/
```

---

# 35. Complete Configuration Template

The maintained configuration is [myodoo11.conf](myodoo11.conf). See [PRODUCTION.md](PRODUCTION.md) for required site-specific changes, worker sizing and HTTPS proxy setup.

---

# 36. Complete systemd Template

The maintained service is [myodoo11.service](myodoo11.service). See [PRODUCTION.md](PRODUCTION.md) for installation, sandbox requirements, journald logging and recovery checks.

---

# 37. Final Verification Checklist

Before considering the installation complete, verify:

- [ ] Ubuntu 20.04 WSL2 is running
- [ ] Dedicated Linux user exists
- [ ] Repository is cloned
- [ ] Python 3.6.15 is installed
- [ ] Python system installation was not replaced
- [ ] Virtual environment exists
- [ ] `setuptools==57.5.0`
- [ ] Odoo requirements installed successfully
- [ ] Additional project Python dependencies installed
- [ ] PostgreSQL running
- [ ] PostgreSQL Odoo user exists
- [ ] Odoo PostgreSQL user has `CREATEDB`
- [ ] Database port matches Odoo config
- [ ] wkhtmltopdf is detected
- [ ] Odoo log directory exists
- [ ] Config file exists
- [ ] Addon paths are correct
- [ ] Odoo starts manually
- [ ] Odoo listens on configured port
- [ ] `curl` returns HTTP 200
- [ ] Browser can access Odoo
- [ ] systemd service starts successfully
- [ ] systemd service is enabled
- [ ] No critical errors appear in the Odoo log

---

# Installation Complete

The Odoo 11 instance should now be accessible using:

```text
http://localhost:ODOO_PORT
```

For future projects, copy this guide and replace:

```text
PROJECT_NAME
DATABASE_PASSWORD
MASTER_PASSWORD
ODOO_PORT
GIT_REPOSITORY
CUSTOM_ADDON_PATHS
```

with the new project's values.
