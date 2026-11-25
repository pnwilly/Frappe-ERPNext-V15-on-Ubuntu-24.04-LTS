# Install Frappe/ERPNext V15 on Ubuntu 24.04 LTS

A step-by-step guide to install **Frappe/ERPNext v15** on **Ubuntu 24.04 LTS**, including **production setup** with Nginx, Supervisor, and SSL.

Requirements checked against Frappe v15.128.0 in November 2026.

---

### System Requirements
Frappe v15 is tested on Python 3.11, Node.js 18 and MariaDB 10.6. This guide installs those versions next to Ubuntu's own, without replacing them.

- **Python 3.11** (from the deadsnakes PPA; Ubuntu's built-in Python 3.12 stays the system default)
- **Node.js 18** and **Yarn 1.12+**
- **Redis 5+**
- **MariaDB 10.6+** (Ubuntu 24.04 ships 10.11) / PostgreSQL
- **wkhtmltopdf 0.12.6.1 (patched qt)**
- **Cron & NGINX** (for production setup)

---

## Installation Steps
Run the following commands, one by one to **install all dependencies and set up Frappe/ERPNext**:


### 1. Update & Install Core Dependencies
    sudo apt update && sudo apt upgrade -y

# In case you face connection issue fetching packages, force IPv4 connection
    echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4

# Add Deadsnakes PPA to get Python 3.11
    sudo apt install -y software-properties-common
    sudo add-apt-repository ppa:deadsnakes/ppa -y
    sudo apt update

# Install Python 3.11 and development tools
Do not change the system `python3` with `update-alternatives`. Ubuntu's own tools, such as `apt` and `add-apt-repository`, need Python 3.12. Bench uses Python 3.11 directly instead.

    sudo apt install -y python3.11 python3.11-venv python3.11-dev

# Install Core Dependencies (excluding Node.js and wkhtmltopdf, installed separately below)
Do not install Ubuntu's `npm` package. It conflicts with the Node.js package installed below, which includes npm.

    sudo apt install -y git python3-dev python3-setuptools python3-pip \
        mariadb-server libmysqlclient-dev redis-server \
        curl xvfb libfontconfig nginx supervisor

# Install patched wkhtmltopdf (required for Frappe PDF generation)
There is no Ubuntu 24.04 build yet. The Ubuntu 22.04 (jammy) build installs on 24.04.

    cd /tmp
    wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb
    sudo apt install -y ./wkhtmltox_0.12.6.1-3.jammy_amd64.deb

# Verify wkhtmltopdf is correctly installed
Output should be: wkhtmltopdf 0.12.6.1 (with patched qt)

    wkhtmltopdf --version

### Secure MariaDB Setup
    sudo systemctl start mariadb
    sudo mysql_secure_installation

### Configure MariaDB for Frappe
This adds a separate settings file, so Ubuntu's own `50-server.cnf` stays unchanged.

    sudo tee /etc/mysql/mariadb.conf.d/99-frappe.cnf > /dev/null <<EOL
    [mysqld]
    character-set-client-handshake = FALSE
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci

    [mysql]
    default-character-set = utf8mb4
    EOL

### Restart MariaDB
    sudo systemctl restart mariadb

### Install Node.js 18 & Yarn
Node.js 18 is what Frappe v15 is tested on. It is past its end of life, so the setup script prints a deprecation warning; the install still works.

    curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
    sudo apt install -y nodejs
    sudo npm install -g yarn

### Install Bench CLI
    python3.11 -m venv ~/.benchenv
    source ~/.benchenv/bin/activate
    pip install frappe-bench


### Initialize Bench & Setup Site
    bench init frappe-bench --frappe-branch version-15 --python python3.11
    cd frappe-bench
    bench set-config -g db_host 127.0.0.1
    bench new-site mysite.local --db-type=mariadb

### Set Active Site
    bench use mysite.local

### Add Site to Hosts File
    echo "127.0.0.1 mysite.local" | sudo tee -a /etc/hosts

### Install ERPNext on Site
    bench get-app erpnext --branch version-15
    bench --site mysite.local install-app erpnext

### Start Development Server
    bench start

### For production setup, run:
Replace [user] with your actual Linux username. `sudo` does not see the bench installed in `~/.benchenv`, so call it by its full path. `setup production` also configures NGINX and Supervisor.

    sudo ~/.benchenv/bin/bench setup production [user]
    sudo supervisorctl restart all
    sudo systemctl restart nginx

>### Notes
>Ubuntu 24.04 installs MariaDB 10.11, which runs Frappe v15. Frappe tests v15 on MariaDB 10.6.
>
>If using PostgreSQL, replace --db-type=mariadb with --db-type=postgres.
