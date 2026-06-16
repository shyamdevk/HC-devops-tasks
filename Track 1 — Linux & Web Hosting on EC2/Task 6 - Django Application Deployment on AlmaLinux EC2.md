# 🚀 Task 6 - Django Application Deployment on AlmaLinux EC2

## 📌 Objective

Deploy the provided Django application on an AlmaLinux EC2 instance using:

* Django
* MariaDB
* Gunicorn
* Apache HTTP Server (Reverse Proxy)
* Let's Encrypt SSL Certificate
* Systemd Service Management

The application must be accessible securely over HTTPS using a custom domain.

---

# 🏗️ Architecture

```text
Internet
    │
    ▼
Apache (HTTPD)
    │
    ▼
Gunicorn (127.0.0.1:8000)
    │
    ▼
Django Application
    │
    ▼
MariaDB Database
```

---

# 👤 Linux User Setup

Created a dedicated Linux user for the application.

```bash
sudo useradd -m django1
sudo passwd django1
```

Verified user home directory:

```text
/home/django1
```

---

# 📦 Application Deployment

Uploaded the provided application archive:

```text
techcake-django1-app.zip
```

Extracted application:

```bash
unzip techcake-django1-app.zip
```

Application location:

```text
/home/django1/techcake-django1-app
```

---

# 🐍 Python Virtual Environment

Created a dedicated Python virtual environment.

```bash
python3 -m venv /home/django1/venv
```

Activated environment:

```bash
source /home/django1/venv/bin/activate
```

Installed dependencies:

```bash
pip install -r requirements.txt
```

---

# 🗄️ MariaDB Configuration

Created dedicated database and user.

```sql
CREATE DATABASE django1;

CREATE USER 'django1_user'@'localhost'
IDENTIFIED BY '********';

GRANT ALL PRIVILEGES ON django1.*
TO 'django1_user'@'localhost';

FLUSH PRIVILEGES;
```

Database Information:

| Item     | Value        |
| -------- | ------------ |
| Database | django1      |
| User     | django1_user |
| Engine   | MariaDB      |

---

# ⚙️ Environment Variable Configuration

Created a dedicated environment file.

Location:

```text
/home/django1/.env
```

Example:

```env
DJANGO_SECRET_KEY=********
DJANGO_DEBUG=false

DJANGO_ALLOWED_HOSTS=django1.shyamdev.nixlabs.in
DJANGO_CSRF_TRUSTED_ORIGINS=https://django1.shyamdev.nixlabs.in

DB_NAME=django1
DB_USER=django1_user
DB_PASSWORD=********
DB_HOST=127.0.0.1
DB_PORT=3306
```

---

# 🔐 Django Settings Configuration

Verified that sensitive values are loaded from environment variables.

Examples:

```python
SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY")
```

```python
DATABASES = {
    "default": {
        "NAME": os.environ.get("DB_NAME"),
        "USER": os.environ.get("DB_USER"),
        "PASSWORD": os.environ.get("DB_PASSWORD"),
    }
}
```

No credentials are hardcoded inside the application.

---

# 🔄 Database Migration

Applied database migrations.

```bash
python manage.py migrate
```

Verified successful migration.

---

# 📁 Static Files

Collected Django static files.

```bash
python manage.py collectstatic --noinput
```

Static files location:

```text
/home/django1/techcake-django1-app/staticfiles
```

Verified CSS, JavaScript, and images load correctly.

---

# 🚀 Gunicorn Configuration

Created a dedicated systemd service.

Location:

```text
/etc/systemd/system/gunicorn.service
```

Configuration highlights:

```ini
[Service]
User=django1
Group=django1

EnvironmentFile=/home/django1/.env
```

Application binding:

```text
127.0.0.1:8000
```

Enabled automatic startup:

```bash
sudo systemctl enable gunicorn
```

Started service:

```bash
sudo systemctl start gunicorn
```

Verification:

```bash
sudo systemctl status gunicorn
```

---

# 🌐 Apache Reverse Proxy Configuration

Created a dedicated Apache VirtualHost configuration.

Location:

```text
/etc/httpd/conf.d/django1.conf
```

Configured reverse proxy:

```apache
ProxyPreserveHost On
ProxyPass / http://127.0.0.1:8000/
ProxyPassReverse / http://127.0.0.1:8000/
```

Configuration validation:

```bash
sudo apachectl configtest
```

Result:

```text
Syntax OK
```

---

# 🌍 DNS Configuration

Created DNS record:

```text
django1.shyamdev.nixlabs.in
```

Mapped DNS record to EC2 public IP.

Verification:

```bash
host django1.shyamdev.nixlabs.in
```

---

# 🔒 SSL Certificate Configuration

Installed Certbot.

Generated SSL certificate:

```bash
sudo certbot --apache \
-d django1.shyamdev.nixlabs.in
```

Configured:

* HTTPS enabled
* HTTP → HTTPS redirect
* Automatic certificate renewal

Verification:

```bash
sudo certbot renew --dry-run
```

---

# 🛡️ SELinux Configuration

Encountered an issue where:

* Apache was running
* Gunicorn was running
* Gunicorn was listening on port 8000
* Website returned HTTP 503

### Root Cause

SELinux prevented Apache (httpd) from communicating with Gunicorn.

### Resolution

Enabled Apache network connections:

```bash
sudo setsebool -P httpd_can_network_connect 1
```

Verification:

```bash
getsebool httpd_can_network_connect
```

Output:

```text
httpd_can_network_connect --> on
```

After enabling the policy, Apache successfully connected to Gunicorn and the website loaded normally.

---

# 🐞 Issues Encountered & Resolutions

## Issue 1: SCP File Upload Failed

### Error

```text
Permission denied (publickey)
```

### Cause

SSH key was not supplied while using SCP.

### Resolution

Used:

```bash
scp -i key.pem techcake-django1-app.zip \
ec2-user@<server-ip>:/tmp/
```

---

## Issue 2: Website Returning 503 Service Unavailable

### Error

```text
503 Service Unavailable
```

### Cause

Apache was unable to communicate with Gunicorn because SELinux network access was blocked.

### Resolution

```bash
sudo setsebool -P httpd_can_network_connect 1
```

Verified:

```bash
getsebool httpd_can_network_connect
```

Output:

```text
httpd_can_network_connect --> on
```

---

## Issue 3: Service Availability After Reboot

### Cause

Gunicorn service must start automatically after server reboot.

### Resolution

Enabled service:

```bash
sudo systemctl enable gunicorn
```

Verified:

```bash
sudo systemctl is-enabled gunicorn
```

Output:

```text
enabled
```

---

# ✅ Final Verification Checklist

* [x] Linux user `django1` created
* [x] Django application deployed
* [x] Python virtual environment configured
* [x] MariaDB database configured
* [x] Environment variables configured
* [x] Database migrations completed
* [x] Static files collected
* [x] Gunicorn service configured
* [x] Gunicorn enabled on boot
* [x] Apache reverse proxy configured
* [x] DNS record created
* [x] HTTPS enabled
* [x] HTTP redirected to HTTPS
* [x] SSL auto-renewal verified
* [x] CSRF trusted origins configured
* [x] Static files served successfully
* [x] SELinux policy configured
* [x] Website accessible over HTTPS

---

# 🌐 Application URL

```text
https://django1.shyamdev.nixlabs.in
```
![Kubernetes](https://github.com/shyamdevk/HC-devops-tasks/blob/images/live2.jpg)
