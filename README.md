# Install Frappe/ERPNext V15 on Ubuntu 24.04 LTS

A step-by-step guide to install Frappe/ERPNext v15 on Ubuntu 24.04 LTS, including production setup with Nginx, Supervisor, and SSL.

---

## System Requirements

Ubuntu 24.04 includes Python 3.12 by default. Ensure the following dependencies are installed:

- Python 3.11+ (Python 3.12 is built-in)
- Node.js 18+
- Redis 5+
- MariaDB 10.8 (recommended) or PostgreSQL
- Yarn 1.12+
- wkhtmltopdf 0.12.5 (with patched Qt)
- Cron & NGINX (for production setup)

---

## Installation Steps

Run the following commands one by one to install all dependencies and set up Frappe/ERPNext.

---

### 1. Update & Install Core Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git python3.11 python3.11-venv python3-dev python3-setuptools python3-pip \
    software-properties-common mariadb-server libmysqlclient-dev redis-server \
    curl npm xvfb libfontconfig nginx supervisor
```

---

### 2. Install Patched wkhtmltopdf (Required for PDF Printing)

```bash
cd /tmp
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb
sudo apt install -y ./wkhtmltox_0.12.6.1-3.jammy_amd64.deb

# Verify installation (should show: wkhtmltopdf 0.12.6 with patched qt)
wkhtmltopdf --version
```

---

### 3. Secure MariaDB

```bash
sudo systemctl start mariadb
sudo mysql_secure_installation
```

---

### 4. Configure MariaDB for Frappe

```bash
sudo tee /etc/mysql/mariadb.conf.d/50-server.cnf > /dev/null <<EOL
[server]
user = mysql
pid-file = /run/mysqld/mysqld.pid
socket = /run/mysqld/mysqld.sock
basedir = /usr
datadir = /var/lib/mysql
tmpdir = /tmp
lc-messages-dir = /usr/share/mysql
bind-address = 127.0.0.1
query_cache_size = 16M
log_error = /var/log/mysql/error.log

[mysqld]
innodb-file-format=barracuda
innodb-file-per-table=1
innodb-large-prefix=1
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
EOL

sudo systemctl restart mariadb
```

---

### 5. Install Node.js 18 and Yarn

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g yarn
```

---

### 6. Install Bench CLI in Python Virtual Environment

```bash
python3.11 -m venv ~/.benchenv
source ~/.benchenv/bin/activate
pip install frappe-bench
```

---

### 7. Initialize Bench & Setup Site

```bash
bench init frappe-bench --frappe-branch version-15
cd frappe-bench
bench set-config -g db_host 127.0.0.1
bench new-site mysite.local --db-type=mariadb
bench use mysite.local
echo "127.0.0.1 mysite.local" | sudo tee -a /etc/hosts
```

---

### 8. Install ERPNext on the Site

```bash
bench get-app erpnext --branch version-15
bench --site mysite.local install-app erpnext
```

---

### 9. Start Development Server

```bash
bench start
```

---

## 🚀 Production Setup

Replace `[user]` with your actual Linux username.

```bash
sudo bench setup production [user]
sudo bench setup nginx
sudo bench setup supervisor
sudo supervisorctl restart all
sudo systemctl restart nginx
```

---

## Notes

- This guide installs MariaDB 10.11 (default in Ubuntu 24.04), but Frappe officially recommends 10.8 for compatibility.
- If using PostgreSQL, replace `--db-type=mariadb` with `--db-type=postgres`.
- For production, `bench setup production` configures Supervisor and NGINX automatically.
