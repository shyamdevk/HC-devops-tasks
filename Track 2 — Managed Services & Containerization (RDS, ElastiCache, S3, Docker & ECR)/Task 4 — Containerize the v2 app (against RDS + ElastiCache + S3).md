
# 🚀 Task 4 — Containerize Django Application

> **TechCake Django App V2 | Docker + Gunicorn + Nginx + AWS Managed Services**

![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker&logoColor=white)
![Django](https://img.shields.io/badge/Django-Application-092E20?logo=django&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-Reverse%20Proxy-009639?logo=nginx&logoColor=white)
![Gunicorn](https://img.shields.io/badge/Gunicorn-WSGI-499848)
![AWS RDS](https://img.shields.io/badge/AWS-RDS-527FFF?logo=amazonaws&logoColor=white)
![Redis](https://img.shields.io/badge/ElastiCache-Redis-DC382D?logo=redis&logoColor=white)
![S3](https://img.shields.io/badge/Amazon-S3-569A31?logo=amazons3&logoColor=white)

---

## 📌 Overview

This task containerizes the existing **TechCake Django V2** application using Docker.

The original application was running directly on the EC2 host using a Python virtual environment and a systemd-managed Gunicorn service.

The application was migrated to:

```text
Docker Container
      │
      └── Gunicorn
             │
             └── Django
````

Nginx remains the public-facing reverse proxy and forwards application traffic to the Docker container.

The existing AWS managed services remain unchanged:

* 🗄️ Amazon RDS → Database
* ⚡ Amazon ElastiCache Redis → Cache
* 🪣 Amazon S3 → Object/media storage
* 🔐 EC2 IAM Role → AWS authentication

---

# 🏗️ Final Architecture

```text
                         INTERNET
                             │
                             ▼
                djangov2.shyamdev.nixlabs.in
                             │
                             ▼
                     DNS Resolution
                             │
                             ▼
                    EC2 Public IP
                             │
                             ▼
                  AWS Security Group
                      │        │
                    HTTP      HTTPS
                     80        443
                      │        │
                      └────┬───┘
                           ▼
                         NGINX
                    SSL Termination
                           │
                           │ proxy_pass
                           ▼
                  127.0.0.1:8001
                           │
                           ▼
                   Docker Container
                           │
                           ▼
                       Gunicorn
                           │
                           ▼
                        Django
                    /      |       \
                   /       |        \
                  ▼        ▼         ▼
                RDS     ElastiCache   S3
              MySQL       Redis
```

---

# 🔄 Request Flow

When a user opens:

```text
https://djangov2.shyamdev.nixlabs.in
```

the request follows this path:

```text
Browser
   │
   ▼
DNS
   │
   ▼
EC2 Public IP
   │
   ▼
Security Group
   │
   ▼
Nginx :443
   │
   │ SSL termination
   ▼
Nginx Reverse Proxy
   │
   │ proxy_pass http://127.0.0.1:8001
   ▼
Docker Host Port :8001
   │
   │ Docker port mapping
   ▼
Container Port :8000
   │
   ▼
Gunicorn
   │
   ▼
Django
   │
   ├──► RDS
   │
   ├──► ElastiCache Redis
   │
   └──► S3
   │
   ▼
Response
   │
   ▼
Gunicorn
   │
   ▼
Docker
   │
   ▼
Nginx
   │
   ▼
Browser
```

---

# 🧩 Application Stack

| Component         | Purpose                   |
| ----------------- | ------------------------- |
| EC2               | Application host          |
| Docker            | Container runtime         |
| Docker Compose    | Container management tool |
| Dockerfile        | Image build instructions  |
| Gunicorn          | Django WSGI server        |
| Django            | Web application           |
| Nginx             | Reverse proxy + HTTPS     |
| Amazon RDS        | MySQL database            |
| ElastiCache Redis | Cache                     |
| Amazon S3         | Object/media storage      |
| IAM Role          | AWS authentication        |

---

# 📂 Project Structure

Final project structure:

```text
techcake-django-app-v2/
│
├── Dockerfile
├── .dockerignore
├── .gitignore
├── manage.py
├── requirements.txt
├── README.md
│
├── techcake_site/
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── ...
│
├── staticfiles/
│
├── congrats/
│
└── venv/
```

> `.env` is intentionally stored outside the project directory at:
>
> `/home/djangov2/.env`

---

# ⚙️ Environment

## Host

```text
OS: AlmaLinux 9.8
Architecture: x86_64
CPU: 2
Memory: ~717 MB
```

## Python

```text
Python 3.9.25
```

## Docker

```text
Docker Engine: 29.6.2
Docker Compose: v5.3.1
```

---

# 1️⃣ Verify Existing Application

Before containerization, the existing application was verified.

### Project

```bash
cd /home/djangov2/techcake-django-app-v2
```

### Python

```bash
python3 --version
```

### pip

```bash
pip3 --version
```

### Gunicorn

```bash
sudo systemctl status gunicorn
```

### Nginx

```bash
sudo systemctl status nginx
```

### S3

```bash
python manage.py s3check
```

The S3 test successfully confirmed:

```text
S3 CHECK PASSED
```

and confirmed:

```text
Credentials were provided by the instance role
(no access keys used).
```

---

# 2️⃣ Install Docker

Docker was installed using:

```bash
sudo dnf install docker -y
```

Start Docker:

```bash
sudo systemctl start docker
```

Enable Docker at boot:

```bash
sudo systemctl enable docker
```

Verify:

```bash
docker --version
```

---

# 3️⃣ Configure Docker Permissions

The EC2 user was added to the Docker group:

```bash
sudo usermod -aG docker ec2-user
```

The current session was refreshed using:

```bash
newgrp docker
```

Docker access was then verified:

```bash
docker ps
```

---

# 4️⃣ Install Docker Compose

Docker Compose plugin was installed using:

```bash
sudo dnf install docker-compose-plugin -y
```

Verify:

```bash
docker compose version
```

Result:

```text
Docker Compose version v5.3.1
```

---

# 5️⃣ Update Python Dependencies

The Django settings use:

```python
from dotenv import load_dotenv
```

Therefore `python-dotenv` was added to `requirements.txt`.

Final requirements:

```text
Django>=4.2,<5.1
gunicorn>=21.2
PyMySQL>=1.1
boto3>=1.34
django-storages>=1.14
redis>=5.0
python-dotenv>=1.0
```

---

# 6️⃣ Environment Variables

The application loads environment variables from:

```text
/home/djangov2/.env
```

Django uses:

```python
load_dotenv("/home/djangov2/.env")
```

The `.env` file contains environment-specific configuration such as:

```text
SECRET_KEY
DEBUG
ALLOWED_HOSTS
CSRF_TRUSTED_ORIGINS
DB_NAME
DB_USER
DB_PASSWORD
DB_HOST
DB_PORT
AWS_STORAGE_BUCKET_NAME
AWS_S3_REGION_NAME
REDIS_URL
```

> ⚠️ Never commit `.env` to GitHub.

---

# 7️⃣ Create `.dockerignore`

`.dockerignore` was created in the project root:

```text
/home/djangov2/techcake-django-app-v2/.dockerignore
```

Contents:

```text
venv/
__pycache__/
*.pyc
*.pyo
*.pyd

.git
.gitignore

README.md

*.sqlite3

.env

staticfiles/

*.log
```

### Why `.env` is ignored

The `.env` file must not be baked into the Docker image.

Instead, it is supplied at runtime.

---

# 8️⃣ Multi-Stage Dockerfile

The Dockerfile uses two stages:

```text
Builder Stage
     │
     ├── Install dependencies
     ├── Create virtual environment
     └── Prepare application
              │
              ▼
Runtime Stage
     │
     ├── Minimal Python image
     ├── Copy virtual environment
     ├── Copy application
     ├── Non-root user
     └── Start Gunicorn
```

## Dockerfile

```dockerfile
# ---------- Stage 1 : Builder ----------
FROM python:3.9-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && \
    apt-get install -y gcc default-libmysqlclient-dev && \
    rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv

ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .

RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY .



# ---------- Stage 2 : Runtime ----------
FROM python:3.9-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN useradd -m appuser

COPY --from=builder /opt/venv /opt/venv

ENV PATH="/opt/venv/bin:$PATH"

COPY --from=builder /app /app

RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "techcake_site.wsgi:application"]
```

---

# 🔍 Dockerfile Explanation

## Builder

```dockerfile
FROM python:3.9-slim AS builder
```

Creates the dependency-building stage.

---

```dockerfile
WORKDIR /app
```

Sets:

```text
/app
```

as the working directory.

---

```dockerfile
COPY requirements.txt .
```

Copies dependencies first to improve Docker layer caching.

---

```dockerfile
RUN pip install ...
```

Installs Python dependencies.

---

```dockerfile
COPY .
```

Copies the application source code.

---

# Runtime Stage

A fresh Python image is used:

```dockerfile
FROM python:3.9-slim
```

Only the required application and Python environment are copied from the builder.

This keeps the final image smaller and avoids carrying unnecessary build dependencies.

---

## Non-root execution

```dockerfile
RUN useradd -m appuser
```

creates a non-root user.

Then:

```dockerfile
USER appuser
```

ensures Gunicorn/Django doesn't run as root.

---

## Gunicorn

```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "techcake_site.wsgi:application"]
```

Gunicorn listens inside the container on:

```text
0.0.0.0:8000
```

---

# 9️⃣ Build Docker Image

Image was built using:

```bash
docker build -t techcake-django:v1 .
```

Verify:

```bash
docker images
```

Result:

```text
techcake-django:v1
```

---

# 🏷️ Image Name vs Website Name

These are completely different things.

### Docker image

```text
techcake-django:v1
```

This is the Docker image repository/tag.

### Website

```text
djangov2.shyamdev.nixlabs.in
```

This is the public DNS/domain name.

The image name does **not** need to match the website name.

---

# 🔟 Run the Container

The container was started with:

```bash
docker run -d \
  --name techcake-test \
  -p 8001:8000 \
  -v /home/djangov2/.env:/home/djangov2/.env:ro \
  techcake-django:v1
```

---

# 🔌 Port Mapping

This is extremely important:

```text
Host
8001
 │
 ▼
Docker
8000
```

The command:

```bash
-p 8001:8000
```

means:

```text
EC2 Host Port 8001
        │
        ▼
Container Port 8000
```

Gunicorn listens on container port `8000`.

Docker exposes it through host port `8001`.

---

# 🔐 Environment File Mount

The command:

```bash
-v /home/djangov2/.env:/home/djangov2/.env:ro
```

means:

```text
EC2:
/home/djangov2/.env
       │
       ▼
Container:
/home/djangov2/.env
```

`ro` means:

```text
Read Only
```

The container can read the configuration but cannot modify the host `.env`.

---

# 1️⃣1️⃣ Verify Container

Check:

```bash
docker ps
```

Expected:

```text
techcake-test
```

with:

```text
0.0.0.0:8001->8000/tcp
```

---

# 1️⃣2️⃣ Troubleshooting: Initial Container Failure

The first container attempt exited with:

```text
Exited (3)
```

The logs showed:

```text
PermissionError: [Errno 13] Permission denied:
'/home/djangov2/.env'
```

### Cause

The container was running as a non-root user, while the mounted `.env` file had restrictive permissions.

### Resolution

The container was subsequently started successfully with the appropriate user/file permissions.

The final container status:

```text
Up
```

---

# 1️⃣3️⃣ Django Verification

Run:

```bash
docker exec techcake-test python manage.py check
```

Expected:

```text
System check identified no issues (0 silenced).
```

This confirms Django loads correctly inside the container.

---

# 1️⃣4️⃣ S3 Verification

Run:

```bash
docker exec techcake-test python manage.py s3check
```

Expected:

```text
S3 CHECK PASSED
```

This confirms:

```text
Docker
   │
   ▼
Django
   │
   ▼
boto3
   │
   ▼
EC2 IAM Role
   │
   ▼
Amazon S3
```

No hardcoded AWS access keys are required.

---

# 1️⃣5️⃣ ALLOWED_HOSTS

The production configuration uses:

```text
ALLOWED_HOSTS=djangov2.shyamdev.nixlabs.in
```

Therefore:

```text
https://djangov2.shyamdev.nixlabs.in
```

works.

But:

```text
http://98.84.55.149:8001
```

returns:

```text
400 Bad Request
```

because the IP address is not in `ALLOWED_HOSTS`.

Similarly:

```text
http://localhost:8001
```

can return:

```text
400 Bad Request
```

because `localhost` is not an allowed host.

This is expected Django security behavior.

---

# 1️⃣6️⃣ Nginx Configuration

Original Nginx proxy:

```nginx
proxy_pass http://127.0.0.1:8000;
```

This pointed to the old host Gunicorn service.

It was changed to:

```nginx
proxy_pass http://127.0.0.1:8001;
```

Now Nginx sends traffic to Docker.

---

# 🌐 Nginx Request Flow

```text
HTTPS Request
      │
      ▼
Nginx :443
      │
      ▼
127.0.0.1:8001
      │
      ▼
Docker
      │
      ▼
Container :8000
      │
      ▼
Gunicorn
      │
      ▼
Django
```

---

# 1️⃣7️⃣ HTTPS / SSL

Nginx terminates HTTPS using the existing Let's Encrypt certificate.

The browser connects using:

```text
HTTPS :443
```

Nginx decrypts the request and proxies internally using HTTP:

```text
127.0.0.1:8001
```

The user still accesses the site securely through HTTPS.

---

# 1️⃣8️⃣ Disable Old Gunicorn Service

After confirming the Docker application was working, the old systemd Gunicorn service was stopped and disabled:

```bash
sudo systemctl stop gunicorn
sudo systemctl disable gunicorn
```

Verify:

```bash
sudo systemctl status gunicorn
```

Expected:

```text
inactive (dead)
disabled
```

### Important

Gunicorn itself was **not removed from the Docker image**.

Gunicorn is still required because Docker runs:

```text
Docker
  └── Gunicorn
        └── Django
```

---

# 🧹 Cleanup

## Remove Docker test image

The temporary Docker test image can be removed:

```bash
docker rmi hello-world:latest
```

---

## Accidental empty settings.py

An empty root-level file was found:

```text
/home/djangov2/techcake-django-app-v2/settings.py
```

It was:

```text
0 bytes
```

The actual Django settings file is:

```text
techcake_site/settings.py
```

The empty root-level file can safely be removed after confirming it remains unused:

```bash
sudo rm /home/djangov2/techcake-django-app-v2/settings.py
```

---

# ⚠️ Files That Must NOT Be Deleted

Keep:

```text
Dockerfile
.dockerignore
requirements.txt
manage.py
techcake_site/
staticfiles/
.gitignore
.env
```

Also keep:

```text
/etc/nginx/conf.d/djangov2.conf
```

and the Let's Encrypt certificate files.

---

# 🔒 Security

## Secrets are NOT baked into the Docker image

`.dockerignore` contains:

```text
.env
```

Therefore `.env` is not copied during:

```bash
docker build
```

Instead, it is mounted at runtime:

```bash
-v /home/djangov2/.env:/home/djangov2/.env:ro
```

---

## AWS Credentials

The application uses the EC2 IAM Role.

S3 verification confirmed:

```text
Credentials were provided by the instance role
(no access keys used).
```

Therefore:

```text
❌ No hardcoded AWS access keys
❌ No hardcoded AWS secret keys
✅ EC2 IAM Role
```

---

# 🔍 Secret Verification

Before pushing to GitHub, check for accidental hardcoded credentials:

```bash
grep -R "AWS_ACCESS_KEY_ID" .
grep -R "AWS_SECRET_ACCESS_KEY" .
grep -R "SECRET_KEY *= *['\"]" .
grep -R "PASSWORD *= *['\"]" .
grep -R "mysql://" .
```

Review the output carefully before committing.

---

# ⚠️ S3 Presigned URLs

The `s3check` command generates temporary presigned URLs.

Example:

```text
https://bucket.s3.amazonaws.com/...
```

Do **not** publish complete presigned URLs in:

* GitHub README
* screenshots
* reports
* public documentation

They can contain temporary authentication information.

---

# 🧪 Final Verification Commands

## Docker

```bash
docker ps
```

---

## Docker logs

```bash
docker logs --tail 50 techcake-test
```

---

## Django

```bash
docker exec techcake-test python manage.py check
```

---

## S3

```bash
docker exec techcake-test python manage.py s3check
```

---

## Local application

```bash
curl -I http://localhost:8001
```

Expected:

```text
HTTP/1.1 200 OK
```

---

## Production website

```bash
curl -I https://djangov2.shyamdev.nixlabs.in
```

Expected:

```text
HTTP/1.1 200 OK
```

---

## Nginx

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

---

## Nginx proxy

```bash
sudo grep -n "proxy_pass" /etc/nginx/conf.d/djangov2.conf
```

Expected:

```text
proxy_pass http://127.0.0.1:8001;
```

---

## Old Gunicorn

```bash
sudo systemctl status gunicorn
```

Expected:

```text
inactive (dead)
disabled
```

---

# 📊 Final Validation Checklist

| Check                             | Status |
| --------------------------------- | -----: |
| Docker installed                  |      ✅ |
| Docker daemon running             |      ✅ |
| Docker Compose installed          |      ✅ |
| Docker permissions configured     |      ✅ |
| `.dockerignore` created           |      ✅ |
| `python-dotenv` added             |      ✅ |
| Multi-stage Dockerfile created    |      ✅ |
| Docker image built                |      ✅ |
| Container running                 |      ✅ |
| Gunicorn running inside container |      ✅ |
| Django system check               |      ✅ |
| RDS configuration                 |      ✅ |
| Redis configuration               |      ✅ |
| S3 connectivity                   |      ✅ |
| IAM Role authentication           |      ✅ |
| Nginx reverse proxy               |      ✅ |
| HTTPS                             |      ✅ |
| Production domain                 |      ✅ |
| Old host Gunicorn disabled        |      ✅ |
| Secrets excluded from image       |      ✅ |

---

# 🏁 Final State

The application is now running using Docker instead of the old native Gunicorn service.

```text
                    ┌─────────────────────┐
                    │       Browser       │
                    └──────────┬──────────┘
                               │
                               │ HTTPS
                               ▼
                    ┌─────────────────────┐
                    │        Nginx        │
                    │      :443 / :80     │
                    └──────────┬──────────┘
                               │
                               │ 127.0.0.1:8001
                               ▼
                    ┌─────────────────────┐
                    │   Docker Container  │
                    │                     │
                    │     Gunicorn        │
                    │        │            │
                    │      Django         │
                    └───────┬─┬─┬────────┘
                            │ │ │
              ┌─────────────┘ │ └─────────────┐
              ▼               ▼               ▼
        ┌──────────┐    ┌────────────┐   ┌──────────┐
        │   RDS    │    │ ElastiCache│   │    S3    │
        │  MySQL   │    │   Redis    │   │ Storage  │
        └──────────┘    └────────────┘   └──────────┘
```

---

# 📝 Important Things to Remember

### Docker image

```text
techcake-django:v1
```

### Container

```text
techcake-test
```

### Host port

```text
8001
```

### Container port

```text
8000
```

### Public domain

```text
djangov2.shyamdev.nixlabs.in
```

### Environment file

```text
/home/djangov2/.env
```

### Nginx configuration

```text
/etc/nginx/conf.d/djangov2.conf
```

### Django settings

```text
techcake_site/settings.py
```

### Gunicorn

Runs **inside Docker**.

### Host Gunicorn

```text
Stopped
Disabled
```

---

# 🔧 Useful Maintenance Commands

## Restart container

```bash
docker restart techcake-test
```

## View logs

```bash
docker logs -f techcake-test
```

## Enter container

```bash
docker exec -it techcake-test /bin/bash
```

## Django shell

```bash
docker exec -it techcake-test python manage.py shell
```

## Django check

```bash
docker exec techcake-test python manage.py check
```

## Database migrations

```bash
docker exec techcake-test python manage.py migrate
```

## List containers

```bash
docker ps -a
```

## List images

```bash
docker images
```

## Docker disk usage

```bash
docker system df
```

---

# 🚨 Troubleshooting Quick Reference

## Container stopped

```bash
docker ps -a
docker logs techcake-test
```

---

## 400 Bad Request

Check:

```bash
grep "^ALLOWED_HOSTS" /home/djangov2/.env
```

Make sure the hostname being used is allowed.

---

## 502 Bad Gateway

Check:

```bash
docker ps
docker logs techcake-test
```

Then verify Nginx:

```bash
sudo nginx -t
```

and:

```bash
sudo grep -n "proxy_pass" /etc/nginx/conf.d/djangov2.conf
```

Expected:

```text
127.0.0.1:8001
```

---

## `.env` permission error

Check:

```bash
ls -l /home/djangov2/.env
```

The non-root user inside the container must be able to read the mounted file.

---

## S3 failure

Run:

```bash
docker exec techcake-test python manage.py s3check
```

Verify the EC2 instance still has the required IAM Role and permissions.

---

## RDS failure

Check the database environment variables:

```text
DB_NAME
DB_USER
DB_PASSWORD
DB_HOST
DB_PORT
```

Do not print the password publicly.

---

# 🎯 Task Completion

## Task 4 — COMPLETED ✅

The Django application was successfully containerized and integrated with the existing AWS managed services.

Final flow:

```text
User
 ↓
DNS
 ↓
EC2
 ↓
Nginx
 ↓
Docker
 ↓
Gunicorn
 ↓
Django
 ↓
RDS / Redis / S3
```

**Containerization successful. 🚀**

````

### One important correction before you push this README

In the Dockerfile section, the line:

```dockerfile
COPY .
````

should actually be:

```dockerfile
COPY . .
```

So make sure your **actual Dockerfile** has:

```dockerfile
# Copy project source
COPY . .
```


