# 🚀 TechCake Django App v2 — Docker Compose Deployment

> **Task 5 — Containerization with Docker Compose, Nginx Reverse Proxy, HTTPS, AWS Managed Services & CloudWatch Logging**

[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Docker Compose](https://img.shields.io/badge/Docker%20Compose-v2-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Nginx](https://img.shields.io/badge/Nginx-Reverse%20Proxy-009639?logo=nginx&logoColor=white)](https://nginx.org/)
[![AWS](https://img.shields.io/badge/AWS-Cloud%20Services-232F3E?logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![HTTPS](https://img.shields.io/badge/HTTPS-Let's%20Encrypt-2ECC71?logo=letsencrypt&logoColor=white)](https://letsencrypt.org/)

---

## 📌 Overview

This project deploys the **TechCake Django application** using Docker Compose on an AlmaLinux EC2 instance.

The application is split into two containers:

- 🐍 **Django + Gunicorn** — application container
- 🌐 **Nginx** — reverse proxy and TLS termination container

The application uses AWS managed services instead of running database and cache containers:

- 🗄️ Amazon RDS — MySQL database
- ⚡ Amazon ElastiCache — Redis cache
- 🪣 Amazon S3 — application storage
- 📊 Amazon CloudWatch Logs — container logging

The deployment is designed so the complete application stack can be started with:

```bash
docker compose up -d
````

---

# 🏗️ Architecture

```text
                         Internet
                            │
                            │ HTTPS :443
                            ▼
                 ┌─────────────────────┐
                 │   Nginx Container   │
                 │   Reverse Proxy     │
                 │   TLS Termination   │
                 └──────────┬──────────┘
                            │
                            │ HTTP :8000
                            ▼
                 ┌─────────────────────┐
                 │  Django Container   │
                 │ Gunicorn :8000      │
                 └───────┬─────┬───────┘
                         │     │
              ┌──────────┘     └─────────────┐
              ▼                              ▼
       ┌─────────────┐                ┌──────────────┐
       │ Amazon RDS  │                │ ElastiCache  │
       │    MySQL    │                │    Redis     │
       └─────────────┘                └──────────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  Amazon S3  │
                  │   Storage   │
                  └─────────────┘


 Docker Container Logs
        │
        ├──────────────► CloudWatch Logs
        │                 └── shyamdev/djangov2/docker
        │
        └── EC2 IAM Role
            (No AWS access keys)
```

---

# 🧩 Components

| Component          | Purpose                         |
| ------------------ | ------------------------------- |
| Django             | Web application                 |
| Gunicorn           | WSGI application server         |
| Nginx              | Reverse proxy + TLS termination |
| Docker             | Container runtime               |
| Docker Compose     | Multi-container orchestration   |
| Amazon RDS         | MySQL database                  |
| Amazon ElastiCache | Redis cache                     |
| Amazon S3          | Object/file storage             |
| Let's Encrypt      | TLS certificate                 |
| CloudWatch Logs    | Container log management        |
| EC2 IAM Role       | AWS authentication              |

---

# 📂 Project Structure

```text
techcake-django-app-v2/
│
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .gitignore
├── requirements.txt
├── manage.py
│
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── djangov2.conf
│
├── techcake_site/
│
├── staticfiles/
│
└── README.md
```

> ⚠️ The actual application directories may contain additional Django files depending on the application version.

---

# 🐳 Docker Compose Configuration

The deployment uses two services:

```text
django
nginx
```

The Django service uses the existing Docker image:

```text
techcake-django:v1
```

The Nginx service uses:

```text
nginx:alpine
```

---

# 🌐 Dedicated Docker Network

A dedicated user-defined bridge network is configured:

```yaml
networks:
  techcake-network:
    name: techcake-network
    driver: bridge
```

Both containers use this network:

```yaml
networks:
  - techcake-network
```

### Why?

The dedicated network allows:

```text
Nginx → Django
```

communication internally without exposing Django's port directly to the internet.

Django exposes port `8000` only to the Docker network:

```yaml
expose:
  - "8000"
```

Nginx is the only container publishing public web ports:

```yaml
ports:
  - "80:80"
  - "443:443"
```

---

# 🔐 HTTPS & Nginx

The host's native Nginx service was stopped so Dockerized Nginx could use ports `80` and `443`.

The host Nginx status was verified as inactive.

The Nginx container handles:

* HTTP traffic
* HTTPS traffic
* TLS termination
* HTTP → HTTPS redirection
* Reverse proxying to Django

---

## 🔑 Let's Encrypt Certificates

The existing Let's Encrypt directory is mounted read-only:

```yaml
- /etc/letsencrypt:/etc/letsencrypt:ro
```

This allows the Nginx container to access the existing certificate files without storing certificates inside the Docker image.

---

# 🔄 Request Flow

A request follows this path:

```text
Client
  │
  ▼
HTTPS :443
  │
  ▼
Nginx Container
  │
  │ TLS termination
  │
  ▼
Django Container :8000
  │
  ├──► RDS MySQL
  │
  ├──► ElastiCache Redis
  │
  └──► S3
```

HTTP requests are redirected to HTTPS.

---

# 🗄️ AWS Managed Database

The application uses **Amazon RDS MySQL**.

There is:

```text
❌ No MySQL container
```

The Django container connects directly to the RDS endpoint through environment variables.

Example configuration structure:

```env
DB_HOST=<RDS_ENDPOINT>
DB_PORT=3306
DB_NAME=<DATABASE_NAME>
DB_USER=<DATABASE_USER>
DB_PASSWORD=<DATABASE_PASSWORD>
```

> 🔒 Never commit the real database password to GitHub.

---

# ⚡ AWS ElastiCache Redis

Redis is provided by **Amazon ElastiCache**.

There is:

```text
❌ No Redis container
```

The application uses the ElastiCache Redis endpoint through the environment configuration.

Example:

```env
REDIS_URL=redis://<ELASTICACHE_ENDPOINT>:6379
```

---

# 🪣 Amazon S3

The application uses Amazon S3 for storage.

Example configuration:

```env
AWS_STORAGE_BUCKET_NAME=<S3_BUCKET_NAME>
AWS_S3_REGION_NAME=us-east-1
```

The application was previously verified using the S3 read/write check.

---

# 🔐 Environment Configuration

The real environment file is stored outside the Git repository:

```text
/home/djangov2/.env
```

Docker Compose loads it using:

```yaml
env_file:
  - /home/djangov2/.env
```

Sensitive values include:

* `SECRET_KEY`
* `DB_PASSWORD`
* Database credentials
* Other environment-specific secrets

These values must **never** be placed directly into:

```text
docker-compose.yml
README.md
GitHub
Screenshots
```

---

# ☁️ CloudWatch Logs

Docker container logging is configured using the AWS `awslogs` logging driver.

## Log Group

```text
shyamdev/djangov2/docker
```

## Django Stream

```text
django
```

## Nginx Stream

```text
nginx
```

Both containers use:

```yaml
logging:
  driver: awslogs
```

---

## 🔑 IAM Authentication

The containers use the EC2 instance's IAM role for CloudWatch authentication.

No AWS access keys are stored inside:

```text
Dockerfile
docker-compose.yml
.env
```

The EC2 instance uses the IAM role:

```text
shyamdev-dev-ec2-role
```

This is safer than storing static AWS credentials inside the application.

---

# ❤️ Health Checks

Health checks are configured for both containers.

## Django

The health check verifies that the Gunicorn application is accepting connections on:

```text
127.0.0.1:8000
```

## Nginx

The health check validates the Nginx configuration:

```bash
nginx -t
```

Verification:

```bash
docker ps
```

Expected:

```text
techcake-django   Up ... (healthy)
techcake-nginx    Up ... (healthy)
```

---

# 🔁 Restart Policy

Both services use:

```yaml
restart: unless-stopped
```

This allows Docker to automatically restart the containers if they stop unexpectedly.

---

# 🚀 Deployment

## Start the complete stack

From the project directory:

```bash
cd /home/djangov2/techcake-django-app-v2
```

Start:

```bash
docker compose up -d
```

This starts:

```text
techcake-django
techcake-nginx
```

---

# 🔍 Verify Docker Compose

```bash
docker compose config
```

The command should complete without YAML or configuration errors.

> ⚠️ `docker compose config` expands values from `.env`. Do not use its output in public screenshots because sensitive values may be displayed.

---

# 📊 Check Containers

```bash
docker compose ps
```

or:

```bash
docker ps
```

Expected:

```text
techcake-django   Up (healthy)
techcake-nginx    Up (healthy)
```

---

# 🌐 Test HTTPS

Test the public application:

```bash
curl -I https://djangov2.shyamdev.nixlabs.in
```

The application should return a successful HTTP response.

Open the application in a browser:

```text
https://djangov2.shyamdev.nixlabs.in
```

Verify:

* HTTPS works
* Certificate is valid
* Application loads
* HTTP redirects to HTTPS

---

# 🔧 Verify Nginx

Run:

```bash
docker exec techcake-nginx nginx -t
```

Expected:

```text
syntax is ok
configuration file /etc/nginx/nginx.conf test is successful
```

---

# 📝 Verify Container Logging

Check the logging driver:

```bash
docker inspect techcake-django --format='{{.HostConfig.LogConfig.Type}}'
docker inspect techcake-nginx --format='{{.HostConfig.LogConfig.Type}}'
```

Expected:

```text
awslogs
awslogs
```

---

# 🧪 Generate Logs

Generate an application request:

```bash
curl -I https://djangov2.shyamdev.nixlabs.in
```

Then refresh the website several times.

Check:

```text
AWS Console
→ CloudWatch
→ Logs
→ Log groups
→ shyamdev/djangov2/docker
```

Verify the:

```text
django
nginx
```

log streams contain recent log events.

---

# 🌐 Verify Dedicated Network

Check:

```bash
docker network ls
```

Expected application network:

```text
techcake-network
```

Inspect it:

```bash
docker network inspect techcake-network
```

The network should contain:

```text
techcake-django
techcake-nginx
```

The network is:

```text
Driver: bridge
```

---

# 🧹 Network Cleanup

During development, Docker Compose initially created project-prefixed networks.

Unused networks were verified to contain no containers and were removed:

```bash
docker network rm techcake-django-app-v2_default
docker network rm techcake-django-app-v2_techcake-network
```

The final dedicated network is:

```text
techcake-network
```

Docker's built-in networks remain:

```text
bridge
host
none
```

These should not be removed.

---

# 🛡️ Security Practices

The deployment follows these practices:

* ✅ Secrets stored in an external `.env` file
* ✅ `.env` should not be committed to GitHub
* ✅ No AWS access keys stored in containers
* ✅ EC2 IAM role used for AWS authentication
* ✅ Let's Encrypt certificates mounted read-only
* ✅ Nginx certificates are not baked into the Docker image
* ✅ Django port `8000` is not publicly published
* ✅ Only ports `80` and `443` are exposed publicly
* ✅ RDS is used instead of a local database container
* ✅ ElastiCache is used instead of a local Redis container
* ✅ S3 is used for application storage

---

# ⚠️ Important GitHub Rules

Before pushing this project to GitHub, verify:

```bash
git status
```

Make sure the real `.env` is not tracked.

Check:

```bash
git ls-files .env
```

If nothing is returned, the `.env` is not tracked.

The repository should contain an environment template instead.

Example:

```text
.env.example
```

or:

```text
.env.template
```

Example:

```env
DEBUG=False

SECRET_KEY=<REDACTED>

DB_HOST=<RDS_ENDPOINT>
DB_PORT=3306
DB_NAME=<DATABASE_NAME>
DB_USER=<DATABASE_USER>
DB_PASSWORD=<REDACTED>

REDIS_URL=redis://<ELASTICACHE_ENDPOINT>:6379

AWS_STORAGE_BUCKET_NAME=<S3_BUCKET_NAME>
AWS_S3_REGION_NAME=us-east-1

ALLOWED_HOSTS=<DOMAIN>
CSRF_TRUSTED_ORIGINS=https://<DOMAIN>
```

---

# 📸 Submission Evidence

The following evidence should be captured for the task submission.

## 1. Docker Compose Status

```bash
docker compose ps
```

Screenshot should show:

```text
techcake-django   Up (healthy)
techcake-nginx    Up (healthy)
```

---

## 2. HTTPS Application

```bash
curl -I https://djangov2.shyamdev.nixlabs.in
```

Also capture the application loading successfully in a browser.

---

## 3. CloudWatch Logs

Navigate to:

```text
CloudWatch
→ Logs
→ Log groups
→ shyamdev/djangov2/docker
```

Capture the log streams and recent log events.

---

## 4. Dedicated Docker Network

```bash
docker network inspect techcake-network
```

The screenshot should show both:

```text
techcake-django
techcake-nginx
```

---

# 📋 Final Verification Checklist

| Requirement                                   | Status |
| --------------------------------------------- | ------ |
| Docker Compose starts the application         | ✅      |
| Django container                              | ✅      |
| Nginx container                               | ✅      |
| Dedicated Docker network                      | ✅      |
| Django + Nginx connected to dedicated network | ✅      |
| Native Nginx stopped                          | ✅      |
| Nginx reverse proxy                           | ✅      |
| HTTPS                                         | ✅      |
| Let's Encrypt certificate                     | ✅      |
| HTTP → HTTPS redirect                         | ✅      |
| RDS MySQL                                     | ✅      |
| No database container                         | ✅      |
| ElastiCache Redis                             | ✅      |
| No Redis container                            | ✅      |
| S3 storage                                    | ✅      |
| Django health check                           | ✅      |
| Nginx health check                            | ✅      |
| Containers report healthy                     | ✅      |
| CloudWatch Logs                               | ✅      |
| `awslogs` driver                              | ✅      |
| EC2 IAM role authentication                   | ✅      |
| No AWS keys in application                    | ✅      |
| Restart policy                                | ✅      |
| Secrets externalized                          | ✅      |

---

# 🏁 Final Result

The TechCake Django application was successfully deployed using **Docker Compose** with a dedicated Docker network and a containerized Nginx reverse proxy.

The final architecture uses:

```text
Docker Compose
├── Nginx Container
│   ├── Reverse Proxy
│   ├── HTTPS
│   └── TLS Termination
│
└── Django Container
    └── Gunicorn
         │
         ├── Amazon RDS
         ├── Amazon ElastiCache
         └── Amazon S3
```

Container logs are delivered to:

```text
CloudWatch Logs
└── shyamdev/djangov2/docker
    ├── django
    └── nginx
```

The deployment is started with a single command:

```bash
docker compose up -d
```

Both containers pass their health checks and communicate through the dedicated:

```text
techcake-network
```

The application is accessible securely through:

```text
https://djangov2.shyamdev.nixlabs.in
```

---

## 🎯 Key Commands to Remember

### Start

```bash
docker compose up -d
```

### Stop

```bash
docker compose down
```

### Status

```bash
docker compose ps
```

### Logs

```bash
docker compose logs
```

### Django logs

```bash
docker logs techcake-django
```

### Nginx logs

```bash
docker logs techcake-nginx
```

### Nginx configuration test

```bash
docker exec techcake-nginx nginx -t
```

### Logging driver

```bash
docker inspect techcake-django --format='{{.HostConfig.LogConfig.Type}}'
docker inspect techcake-nginx --format='{{.HostConfig.LogConfig.Type}}'
```

### Network

```bash
docker network inspect techcake-network
```

### HTTPS test

```bash
curl -I https://djangov2.shyamdev.nixlabs.in
```

---

> **Status: ✅ Task 5 Completed**
>
> Docker Compose, dedicated networking, containerized Nginx, HTTPS, AWS managed services, health checks, IAM-based CloudWatch logging, and application verification have been completed.

```
```
