# Install Frappe/ERPNext V15 on Ubuntu 24.04 LTS

A step-by-step guide to install **Frappe/ERPNext v15** on **Ubuntu 24.04 LTS**, including **production setup** with Nginx, Supervisor, and SSL.

---

### System Requirements
Ubuntu 24.04 includes **Python 3.12** by default. Ensure the following dependencies are installed:

- **Python 3.11+** (Python 3.12 is built-in)
- **Node.js 18+**
- **Redis 5+**
- **MariaDB 10.8 (recommended) / PostgreSQL**
- **Yarn 1.12+**
- **wkhtmltopdf 0.12.5 (patched qt)**
- **Cron & NGINX** (for production setup)

---

## Installation Steps
Run the following commands, one by one to **install all dependencies and set up Frappe/ERPNext**:


### 1. Update & Install Core Dependencies
    sudo apt update && sudo apt upgrade -y

# Add Deadsnakes PPA to get newer Python versions
    sudo apt update
    sudo apt install -y software-properties-common
    sudo add-apt-repository ppa:deadsnakes/ppa -y
    sudo apt update

# Install Python 3.11 and development tools
    sudo apt install -y python3.11 python3.11-venv python3.11-dev

# Set Python 3.11 as the default python3 (optional but recommended)
    sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 1
    sudo update-alternatives --config python3
    
# Install Core Dependencies (excluding wkhtmltopdf, installed separately below)
    sudo apt install -y git python3.11 python3.11-venv python3-dev python3-setuptools python3-pip \
        software-properties-common mariadb-server libmysqlclient-dev redis-server \
        curl npm xvfb libfontconfig nginx supervisor
    
# Install patched (for Jammy) wkhtmltopdf (required for Frappe PDF generation)
    cd /tmp
    wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
    sudo apt install -y ./wkhtmltox_0.12.6.1-2.jammy_amd64.deb

# In case you face connection issue fetching additional packages, force IPv4 connection
    echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4

# Verify wkhtmltopdf is correctly installed
Output should be: wkhtmltopdf 0.12.6 (with patched qt)
    
    wkhtmltopdf --version

### Secure MariaDB Setup
    sudo systemctl start mariadb
    sudo mysql_secure_installation

### Configure MariaDB for Frappe
This directly edits the 50-server.cnf file using tee.
    
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

### Restart MariaDB
    sudo systemctl restart mariadb

### Install Node.js 18 & Yarn
    curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
    sudo apt install -y nodejs
    sudo npm install -g yarn

### Install Bench CLI
    python3.11 -m venv ~/.benchenv
    source ~/.benchenv/bin/activate
    pip install frappe-bench


### Initialize Bench & Setup Site
    bench init frappe-bench --frappe-branch version-15
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
Replace [user] with your actual Linux username
    
    sudo bench setup production [user]
    sudo bench setup nginx
    sudo bench setup supervisor
    sudo supervisorctl restart all
    sudo systemctl restart nginx

>### Notes
>This guide installs MariaDB 10.11, but Frappe recommends 10.8 for compatibility.
>
>If using PostgreSQL, replace --db-type=mariadb with --db-type=postgres.
>
>For production, use bench setup production frappe to configure NGINX & Supervisor.

