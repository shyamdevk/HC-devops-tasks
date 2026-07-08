# Django Track 2 Deployment Documentation

## Overview

This repository documents the complete deployment of the Django Track 2 application on an Amazon EC2 instance using Nginx, Gunicorn, MariaDB, Redis, and Let's Encrypt SSL. The deployment follows secure configuration practices by using environment variables, systemd service management, and restricted network access.

---

# Architecture

```
Internet
     │
     ▼
 Nginx (80/443)
     │
     ▼
 Gunicorn (127.0.0.1:8000)
     │
     ▼
 Django Application
     │
     ├──────────────► MariaDB
     │
     └──────────────► Redis
```

---

# Technologies Used

| Technology | Purpose |
|------------|---------|
| Amazon EC2 | Application Server |
| AlmaLinux 9 | Operating System |
| Python 3.9 | Django Runtime |
| Django | Web Framework |
| Gunicorn | WSGI Application Server |
| Nginx | Reverse Proxy |
| MariaDB | Database |
| Redis | Cache / Visit Counter |
| Let's Encrypt | SSL Certificate |
| Systemd | Gunicorn Service Management |
| AWS Systems Manager | Remote Management |

---

# Deployment Steps

---

## Step 1 — Launch EC2 Instance

Created an EC2 instance running AlmaLinux 9.

Configured:

- Instance Name
- Security Groups
- IAM Role
- Elastic IP (if required)

Verified SSH access.

---

## Step 2 — Update Server

Updated all installed packages.

```bash
sudo dnf update -y
```

This ensures the operating system is running the latest security updates.

---

## Step 3 — Install Required Packages

Installed all required software.

```bash
sudo dnf install python3 python3-pip python3-devel gcc nginx mariadb-server redis git -y
```

Installed Certbot for SSL.

---

## Step 4 — Create Application User

Created a dedicated Linux user.

```bash
sudo useradd -m djangov2
```

This isolates the application from the root account.

---

## Step 5 — Copy Project

Copied the Django project into

```
/home/djangov2/
```

Changed ownership.

```bash
sudo chown -R djangov2:djangov2 /home/djangov2
```

---

## Step 6 — Create Python Virtual Environment

Switched to the application user.

```bash
su - djangov2
```

Created virtual environment.

```bash
python3 -m venv venv
```

Activated it.

```bash
source venv/bin/activate
```

Installed project dependencies.

```bash
pip install -r requirements.txt
```

---

## Step 7 — Configure Environment Variables

Created the application environment file.

```
/home/djangov2/.env
```

Configured:

- SECRET_KEY
- DEBUG
- ALLOWED_HOSTS
- CSRF_TRUSTED_ORIGINS
- Database Credentials
- Redis URL

The application now loads all sensitive configuration from the environment instead of hardcoded values.

---

## Step 8 — Configure MariaDB

Started MariaDB.

```bash
sudo systemctl enable --now mariadb
```

Created:

- Database
- Database User
- Database Password

Granted required permissions.

Verified database connectivity.

---

## Step 9 — Configure Redis

Started Redis.

```bash
sudo systemctl enable --now redis
```

Configured Django to use Redis as its cache backend.

Verified Redis connectivity.

```bash
redis-cli ping
```

Expected output:

```
PONG
```

---

## Step 10 — Apply Database Migrations

Activated virtual environment.

Executed:

```bash
python manage.py migrate
```

Created database tables.

---

## Step 11 — Collect Static Files

Collected static assets.

```bash
python manage.py collectstatic
```

Verified successful collection.

---

## Step 12 — Configure Gunicorn

Created a systemd service.

```
/etc/systemd/system/gunicorn.service
```

Configured:

- Working Directory
- Virtual Environment
- Environment File
- Restart Policy

Reloaded systemd.

```bash
sudo systemctl daemon-reload
```

Enabled Gunicorn.

```bash
sudo systemctl enable gunicorn
```

Started Gunicorn.

```bash
sudo systemctl start gunicorn
```

Verified service status.

---

## Step 13 — Configure Nginx

Created an Nginx virtual host.

Configured:

- Reverse Proxy
- Static Files
- HTTPS Redirection

Verified configuration.

```bash
sudo nginx -t
```

Restarted Nginx.

---

## Step 14 — Configure SSL

Generated SSL certificate.

Configured HTTPS using Let's Encrypt.

Verified certificate installation.

Confirmed:

- HTTPS works
- HTTP redirects to HTTPS

---

## Step 15 — Configure Security Groups

Configured inbound rules.

Allowed:

| Port | Source |
|-------|--------|
| 80 | Public |
| 443 | Public |
| 22 | Specific IP |
| Office IP | Restricted |
| VPN IP | Restricted |

Verified no database or application ports are publicly accessible.

---

## Step 16 — Configure Redis Visit Counter

Configured Django cache.

Implemented Redis-backed visit counter.

Verified:

- Counter increments on refresh.
- Counter stored in Redis.

![Kubernetes](https://github.com/shyamdevk/HC-devops-tasks/blob/images/CounterIncrementLog.jpg)
![Kubernetes](https://github.com/shyamdevk/HC-devops-tasks/blob/images/WebsiteWithCounterIncrement.jpg)

---

## Step 17 — Configure AWS Systems Manager

Configured Session Manager.

Verified remote access without requiring public SSH access.

---

## Step 18 — Remove Unused Resources

Performed cleanup.

Removed:

- Unused ZIP files
- Temporary files
- Duplicate virtual environment

Verified only one virtual environment remains.

```
/home/djangov2/techcake-django-app-v2/venv
```

---

## Step 19 — Final Security Verification

Verified:

- SELinux Enforcing
- HTTPS working
- Gunicorn running
- Redis running
- MariaDB running
- Environment variables loaded from `.env`
- No hardcoded secrets
- SSH restricted to specific IP
- Application accessible over HTTPS

---

# Verification Commands

## Gunicorn

```bash
sudo systemctl status gunicorn
```

---

## Nginx

```bash
sudo systemctl status nginx
```

---

## MariaDB

```bash
sudo systemctl status mariadb
```

---

## Redis

```bash
sudo systemctl status redis
```

---

## SELinux

```bash
getenforce
```

Expected:

```
Enforcing
```

---

## Redis

```bash
redis-cli ping
```

Expected:

```
PONG
```

---

## HTTPS

```bash
curl -I https://djangov2.shyamdev.nixlabs.in
```

Expected:

```
HTTP/1.1 200 OK
```

![Kubernetes](https://github.com/shyamdevk/HC-devops-tasks/blob/images/LiveWebsitewithPaddleLock.jpg)

---

## Virtual Environment

```bash
find /home/djangov2 -type d -name "venv"
```

Expected:

```
/home/djangov2/techcake-django-app-v2/venv
```

---

# Project Structure

```
/home/djangov2
│
├── .env
│
└── techcake-django-app-v2
    ├── manage.py
    ├── requirements.txt
    ├── techcake_site
    ├── congrats
    ├── staticfiles
    └── venv
```

---

# Security Measures Implemented

- Dedicated Linux application user
- Single Python virtual environment
- Gunicorn managed by systemd
- Environment variables stored separately
- No hardcoded credentials
- HTTPS enforced
- Redis accessible only locally
- Gunicorn bound to localhost
- SELinux enabled
- SSH restricted to trusted IP addresses
- AWS Systems Manager enabled for remote administration

---

# Final Result

The Django application was successfully deployed on Amazon EC2 with a secure production-style configuration. The deployment uses Nginx as the reverse proxy, Gunicorn as the WSGI server, MariaDB as the relational database, Redis for caching, and Let's Encrypt for HTTPS. Configuration is managed through environment variables, services are controlled using systemd, unnecessary files have been removed, and the server has been verified for functionality and security.
