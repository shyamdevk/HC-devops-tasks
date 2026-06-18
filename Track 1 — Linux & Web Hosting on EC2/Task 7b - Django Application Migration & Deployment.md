# 🚀 Task 7B - Django Application Migration & Deployment

## 📌 Overview

This task involved migrating the existing Django application from **django1** to **django2**, creating an independent deployment environment, configuring a dedicated database, setting up Gunicorn and Apache, securing the application with HTTPS, and managing configuration through environment variables.

---

# 🌐 Live Application

**URL:** https://django2.shyamdev.nixlabs.in

---

# 🏗️ Architecture

```text
Internet
    │
    ▼
Apache (HTTPS)
    │
    ▼
Gunicorn (127.0.0.1:8001)
    │
    ▼
Django Application
    │
    ▼
MariaDB Database (django2)
```

---

# 📋 Prerequisites

* AlmaLinux 9 EC2 Instance
* Apache HTTP Server
* MariaDB Server
* Python 3
* Virtual Environment Support
* Certbot
* DNS Access

---

# 👤 Step 1: Create Django User

Created a dedicated Linux user for the new deployment.

```bash
sudo useradd -m django2
sudo passwd django2
```

Verify:

```bash
id django2
```

---

# 📦 Step 2: Copy Django Application

Copied the existing Django project to the new user directory.

```bash
sudo cp -a /home/django1/techcake-django1-app /home/django2/
sudo chown -R django2:django2 /home/django2/techcake-django1-app
```

Project Location:

```text
/home/django2/techcake-django1-app
```

---

# 🐍 Step 3: Create Virtual Environment

Created an isolated Python environment.

```bash
sudo -u django2 python3 -m venv /home/django2/venv
```

Activate:

```bash
source /home/django2/venv/bin/activate
```

---

# 📚 Step 4: Install Application Dependencies

Installed all required Python packages.

```bash
pip install -r /home/django2/techcake-django1-app/requirements.txt
```

Installed Components:

* Django
* Gunicorn
* PyMySQL
* SQLParse
* Other required dependencies

---

# 🗄️ Step 5: Create Database

Created a dedicated database and database user.

```sql
CREATE DATABASE django2;

CREATE USER 'django2_user'@'localhost'
IDENTIFIED BY '********';

GRANT ALL PRIVILEGES ON django2.* TO 'django2_user'@'localhost';

FLUSH PRIVILEGES;
```

---

# 🔄 Step 6: Migrate Existing Data

Exported existing data:

```bash
mysqldump -u django1_user -p django1 > django1.sql
```

Imported into django2:

```bash
mysql -u django2_user -p django2 < django1.sql
```

Verification:

```bash
mysql -u django2_user -p django2
```

---

# ⚙️ Step 7: Configure Environment Variables

Created:

```text
/home/django2/.env
```

Contents:

```env
DJANGO_SECRET_KEY=my-secret-key
DJANGO_DEBUG=false

DJANGO_ALLOWED_HOSTS=django2.shyamdev.nixlabs.in
DJANGO_CSRF_TRUSTED_ORIGINS=https://django2.shyamdev.nixlabs.in

DB_NAME=django2
DB_USER=django2_user
DB_PASSWORD=********
DB_HOST=127.0.0.1
DB_PORT=3306
```

Secure Permissions:

```bash
chmod 600 /home/django2/.env
chown django2:django2 /home/django2/.env
```

---

# 🔐 Step 8: Configure Django Settings

Verified that configuration values are loaded from environment variables.

Example:

```python
SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY")

DATABASES = {
    "default": {
        "NAME": os.environ.get("DB_NAME"),
        "USER": os.environ.get("DB_USER"),
        "PASSWORD": os.environ.get("DB_PASSWORD"),
        "HOST": os.environ.get("DB_HOST"),
        "PORT": os.environ.get("DB_PORT"),
    }
}
```

No secrets are hardcoded.

---

# 🧩 Step 9: Apply Database Migrations

```bash
python manage.py migrate
```

Verification:

```bash
python manage.py migrate --plan
```

Output:

```text
No planned migration operations.
```

---

# 📂 Step 10: Collect Static Files

```bash
python manage.py collectstatic --noinput
```

Static files stored in:

```text
/home/django2/techcake-django1-app/staticfiles
```

---

# 🔥 Step 11: Configure Gunicorn Service

Created:

```text
/etc/systemd/system/gunicorn-django2.service
```

Configuration:

```ini
[Unit]
Description=Gunicorn for TechCake Django App
After=network.target

[Service]
User=django2
Group=django2
WorkingDirectory=/home/django2/techcake-django1-app

EnvironmentFile=/home/django2/.env

ExecStart=/bin/bash -lc 'cd /home/django2/techcake-django1-app && source /home/django2/venv/bin/activate && gunicorn --bind 127.0.0.1:8001 techcake_site.wsgi:application'

Restart=always

[Install]
WantedBy=multi-user.target
```

Reload:

```bash
sudo systemctl daemon-reload
```

Enable:

```bash
sudo systemctl enable gunicorn-django2
```

Start:

```bash
sudo systemctl restart gunicorn-django2
```

Verify:

```bash
sudo systemctl status gunicorn-django2
```

---

# 🌍 Step 12: Configure Apache Reverse Proxy

Created VirtualHost configuration.

```apache
<VirtualHost *:80>
    ServerName django2.shyamdev.nixlabs.in

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:8001/
    ProxyPassReverse / http://127.0.0.1:8001/
</VirtualHost>
```

Verify:

```bash
sudo apachectl configtest
```

Expected:

```text
Syntax OK
```

Restart Apache:

```bash
sudo systemctl restart httpd
```

---

# 🌐 Step 13: Configure DNS

Created DNS record:

```text
Host : django2
Type : A
Value: EC2 Public IP
```

Result:

```text
django2.shyamdev.nixlabs.in
```

---

# 🔒 Step 14: Configure HTTPS

Generated SSL certificate.

```bash
sudo certbot --apache -d django2.shyamdev.nixlabs.in
```

Configured:

* HTTPS Access
* Automatic HTTP → HTTPS Redirect

Verify:

```bash
curl -I https://django2.shyamdev.nixlabs.in
```

Result:

```text
HTTP/1.1 200 OK
```

---

# ♻️ Step 15: Verify Certificate Renewal

```bash
sudo certbot renew --dry-run
```

Successful renewal simulation confirms automatic renewal is configured.

---

# ✅ Validation

### Gunicorn

```bash
sudo systemctl status gunicorn-django2
```

Result:

```text
active (running)
```

### Enabled On Boot

```bash
sudo systemctl is-enabled gunicorn-django2
```

Result:

```text
enabled
```

### Environment File Loaded

```bash
sudo systemctl show gunicorn-django2 | grep EnvironmentFile
```

Result:

```text
EnvironmentFiles=/home/django2/.env
```

### Database Connectivity

```bash
mysql -u django2_user -p django2
```

Connection successful.

### Gunicorn Port Binding

```bash
sudo ss -tulpn | grep 8001
```

Result:

```text
127.0.0.1:8001
```

---

# 🛡️ Security Improvements

✅ Secrets stored in `.env`

✅ Environment file permissions restricted

✅ No hardcoded database credentials

✅ No hardcoded Django secret key

✅ Gunicorn accessible only through localhost

✅ HTTPS enabled

✅ Automatic certificate renewal configured

---

# 📸 Evidence Captured

* Live HTTPS Website
* Gunicorn Service Running
* Gunicorn Service Enabled
* Environment File Configuration
* SSL Certificate Validation

---

# 🎯 Final Result

Successfully migrated the Django application from **django1** to **django2** with:

* Dedicated Linux User
* Dedicated Virtual Environment
* Dedicated MariaDB Database
* Dedicated Gunicorn Service
* Apache Reverse Proxy
* Environment Variable Configuration
* HTTPS with Let's Encrypt
* Automatic Startup on Reboot

### 🌐 Live URL

https://django2.shyamdev.nixlabs.in
