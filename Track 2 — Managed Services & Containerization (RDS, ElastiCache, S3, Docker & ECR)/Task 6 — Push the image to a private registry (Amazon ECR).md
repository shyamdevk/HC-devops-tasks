
# 🚀 TechCake Django App v2 — Amazon ECR Deployment

![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazonaws)
![Amazon ECR](https://img.shields.io/badge/Amazon-ECR-Private%20Registry-orange?logo=amazonaws)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker)
![Django](https://img.shields.io/badge/Django-Application-092E20?logo=django)
![Nginx](https://img.shields.io/badge/Nginx-Reverse%20Proxy-009639?logo=nginx)
![Amazon RDS](https://img.shields.io/badge/Amazon-RDS-blue?logo=amazonaws)
![ElastiCache](https://img.shields.io/badge/Amazon-ElastiCache-red?logo=amazonaws)
![Amazon S3](https://img.shields.io/badge/Amazon-S3-orange?logo=amazons3)

> **Task 6 — Push the image to a private registry (Amazon ECR)**

---

## 📌 Overview

This task demonstrates how the containerized **TechCake Django Application v2** is stored in a private **Amazon Elastic Container Registry (ECR)** repository and deployed by pulling the image from ECR.

The deployment was also validated against the managed AWS services used by the application:

- ☁️ Amazon RDS
- ⚡ Amazon ElastiCache / Redis
- 🪣 Amazon S3
- 🔐 IAM Instance Role
- 🐳 Docker
- 🌐 Nginx
- 🔒 HTTPS / Let's Encrypt

The main objective is to prove that the application can be recreated from the private ECR registry rather than depending on a locally built Docker image.

---

# 🎯 Task Objective

The task requires:

- Create a **private ECR repository**
- Authenticate Docker with ECR
- Tag the application image with:
  - a versioned tag (`v2`)
  - `latest`
- Push both tags to ECR
- Pull the image from ECR on a clean state/host
- Run the application using the ECR image
- Verify:
  - Amazon RDS
  - Amazon ElastiCache
  - Amazon S3 via IAM instance role
- Explain why using only `latest` is dangerous

---

# 🏗️ Architecture

```text
                         ┌──────────────────────┐
                         │    Amazon ECR         │
                         │  Private Repository   │
                         │                      │
                         │  ┌─────┐  ┌───────┐ │
                         │  │ v2  │  │latest │ │
                         │  └──┬──┘  └───┬───┘ │
                         └─────┼──────────┼─────┘
                               │          │
                               │ docker pull
                               ▼
                    ┌────────────────────────┐
                    │       EC2 Host         │
                    │                        │
                    │   Docker Compose       │
                    │                        │
                    │ ┌───────────────────┐  │
                    │ │   Django/Gunicorn │  │
                    │ │  techcake-django  │  │
                    │ └─────────┬─────────┘  │
                    │           │            │
                    │ ┌─────────▼─────────┐  │
                    │ │      Nginx        │  │
                    │ │  techcake-nginx   │  │
                    │ └─────────┬─────────┘  │
                    └───────────┼────────────┘
                                │
                         HTTPS / Domain
                                │
                                ▼
                 djangov2.shyamdev.nixlabs.in


       ┌────────────────────── AWS Managed Services ──────────────────────┐
       │                                                                  │
       │   Amazon RDS          ElastiCache Redis             Amazon S3     │
       │       │                       │                         │         │
       │       └────────────── Django Application ──────────────┘         │
       │                                                                  │
       └──────────────────────────────────────────────────────────────────┘
````

---

# 📁 Project Location

The application project is maintained under:

```text
/home/djangov2/techcake-django-app-v2/
```

Environment variables and secrets are stored separately:

```text
/home/djangov2/.env
```

### Important

The `.env` file must **not** be stored inside the Git repository.

---

# ☁️ AWS Configuration

| Resource           | Configuration                                  |
| ------------------ | ---------------------------------------------- |
| AWS Region         | `us-east-1`                                    |
| ECR Repository     | `shyamdev-techcake-django`                     |
| ECR Registry       | `084828581506.dkr.ecr.us-east-1.amazonaws.com` |
| ECR Image          | `shyamdev-techcake-django`                     |
| Version Tag        | `v2`                                           |
| Floating Tag       | `latest`                                       |
| S3 Bucket          | `shyamdev-dev-s3`                              |
| EC2 IAM Role       | `shyamdev-dev-ec2-role`                        |
| Application Domain | `djangov2.shyamdev.nixlabs.in`                 |

> Do not treat the AWS account ID, repository name, bucket name, or domain as credentials. Never publish passwords, secret keys, session tokens, or private `.env` contents.

---

# 🐳 Docker Components

The deployment currently uses two application containers.

## 1. Django Container

```text
Container:
techcake-django
```

Image:

```text
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:v2
```

Application server:

```text
Gunicorn
```

Port:

```text
8000
```

The port is exposed internally to the Docker network and is not directly published to the Internet.

---

## 2. Nginx Container

```text
Container:
techcake-nginx
```

Image:

```text
nginx:alpine
```

Published ports:

```text
80
443
```

Nginx provides:

* HTTP → HTTPS redirect
* TLS termination
* Reverse proxy
* Static file serving

---

# 🌐 Docker Network

Application network:

```text
techcake-network
```

Network type:

```text
bridge
```

Both containers are connected to this network.

Check:

```bash
docker network ls
```

Inspect:

```bash
docker network inspect techcake-network
```

---

# 🪣 Static Files Volume

The current deployment uses one named Docker volume for Django static files:

```text
techcake-django-app-v2_static_volume
```

Purpose:

```text
Django
   │
   │ collectstatic
   ▼
/app/staticfiles
   │
   │ shared Docker volume
   ▼
Nginx
   │
   ▼
/static/
```

This was added because the fresh ECR deployment did not contain collected static files and the Django Admin initially loaded without CSS.

### Current volume

```bash
docker volume ls
```

Expected:

```text
techcake-django-app-v2_static_volume
```

Do **not** delete this volume while the current Nginx static-file configuration depends on it.

---

# 🔐 Environment Variables and Secrets

Application secrets are loaded from:

```text
/home/djangov2/.env
```

Docker Compose uses:

```yaml
env_file:
  - /home/djangov2/.env
```

### Never commit:

```text
.env
```

The `.gitignore` contains:

```gitignore
.env
```

Other ignored files include:

```gitignore
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/
env/

staticfiles/
*.sqlite3
*.log
```

---

# 🛡️ IAM Role

The EC2 instance uses the IAM role:

```text
shyamdev-dev-ec2-role
```

AWS credentials are obtained through the EC2 Instance Metadata Service.

The application does **not** require hardcoded AWS access keys.

The S3 validation confirmed:

```text
Credentials were provided by the instance role (no access keys used).
```

This is the preferred approach for AWS workloads running on EC2.

---

# 🔑 ECR Authentication

Authenticate Docker with ECR:

```bash
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin \
084828581506.dkr.ecr.us-east-1.amazonaws.com
```

Expected:

```text
Login Succeeded
```

### Docker credential warning

Docker may display:

```text
WARNING! Your credentials are stored unencrypted in
'/home/ec2-user/.docker/config.json'.
```

This warning refers to the local Docker credential configuration.

It does **not** mean the ECR image itself contains credentials.

---

# 🏷️ Image Tagging Strategy

The image uses two tags:

```text
v2
latest
```

Example:

```bash
docker tag techcake-django:v1 \
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:v2
```

And:

```bash
docker tag techcake-django:v1 \
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:latest
```

---

# 📌 Why Use Both `v2` and `latest`?

## Versioned Tag

```text
v2
```

A versioned tag identifies a specific application release.

Example:

```text
v1
v2
v3
v4
```

This allows a deployment to explicitly request a known version.

---

## `latest`

```text
latest
```

The `latest` tag normally points to the most recently designated image.

It is convenient for development, but it should not be the only deployment reference.

---

# ⚠️ Why `latest` Alone Is Dangerous

Consider:

```text
production → latest
```

Today:

```text
latest → version A
```

Tomorrow:

```text
latest → version B
```

The same deployment command can therefore result in a different application version.

This makes:

* Rollbacks harder
* Debugging harder
* Auditing harder
* Reproducibility harder

A safer deployment can use:

```text
shyamdev-techcake-django:v2
```

because the version is explicit.

---

# 📤 Push Image to ECR

Push the versioned image:

```bash
docker push \
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:v2
```

Push the latest tag:

```bash
docker push \
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:latest
```

Successful push output contains:

```text
v2: digest: sha256:...
```

and:

```text
latest: digest: sha256:...
```

---

# 🖥️ Verify Images in ECR

Open:

```text
AWS Console
→ ECR
→ Repositories
→ shyamdev-techcake-django
→ Images
```

Expected tags:

```text
latest
v2
```

The ECR console may display:

```text
latest, v2
```

on the same row.

### Why?

Both tags currently point to the same image digest.

Conceptually:

```text
latest ─┐
        ├── sha256:c47a52...
v2 ─────┘
```

This is completely valid.

They do **not** need to appear as separate rows when they reference the same image.

---

# 🧹 Clean-State Deployment Test

The task requires proving that the image can be pulled from ECR instead of relying on a locally built image.

## Step 1 — Stop the application

From:

```text
/home/djangov2/techcake-django-app-v2/
```

run:

```bash
docker compose down
```

---

## Step 2 — Remove the Local ECR Image

```bash
docker rmi \
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:v2
```

This removes the local copy.

---

## Step 3 — Pull From ECR

```bash
docker compose pull
```

Docker retrieves the image from the private ECR repository.

---

## Step 4 — Start the Application

```bash
docker compose up -d
```

---

## Step 5 — Verify Containers

```bash
docker ps
```

Expected containers:

```text
techcake-django
techcake-nginx
```

Both should be healthy/running.

---

# 🔍 Verify the Running Image

Check:

```bash
docker inspect techcake-django \
--format '{{.Config.Image}}'
```

Expected:

```text
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:v2
```

This is an important proof that the running Django container uses the ECR image.

---

# 🌐 Application Verification

Test the application:

```bash
curl -I http://localhost
```

Expected:

```text
HTTP/1.1 301 Moved Permanently
```

The application redirects HTTP to HTTPS.

Test HTTPS:

```bash
curl -I https://djangov2.shyamdev.nixlabs.in
```

---

# 🔒 HTTPS Configuration

Domain:

```text
djangov2.shyamdev.nixlabs.in
```

Nginx uses the Let's Encrypt certificate:

```text
/etc/letsencrypt/live/djangov2.shyamdev.nixlabs.in/fullchain.pem
```

and:

```text
/etc/letsencrypt/live/djangov2.shyamdev.nixlabs.in/privkey.pem
```

Certificate configuration is mounted into the Nginx container.

---

# 🗄️ Amazon RDS Verification

The Django application uses Amazon RDS as the database backend.

Verify database connectivity:

```bash
docker compose exec django python manage.py shell
```

Then:

```python
from django.db import connection

connection.ensure_connection()

print("RDS Connected")
```

Expected:

```text
RDS Connected
```

Exit:

```python
exit()
```

---

# ⚡ Amazon ElastiCache / Redis Verification

Verify Redis cache connectivity:

```bash
docker compose exec django python manage.py shell
```

Run:

```python
from django.core.cache import cache

cache.set("test", "ok")

print(cache.get("test"))
```

Expected:

```text
ok
```

This confirms that the Django application can communicate with the configured Redis cache.

Exit:

```python
exit()
```

---

# 🪣 Amazon S3 Verification

The application includes a custom Django management command:

```text
s3check
```

Run:

```bash
docker compose exec django python manage.py s3check
```

Expected:

```text
S3 CHECK PASSED — wrote and read back 60 bytes.
```

The command:

1. Writes a test object to S3.
2. Reads the object back.
3. Confirms the data is accessible.
4. Uses the EC2 instance IAM role instead of hardcoded AWS credentials.

Example object path:

```text
s3check/techcake-<random-id>.txt
```

---

# 🔐 S3 IAM Role Verification

The S3 check also confirmed:

```text
Credentials were provided by the instance role (no access keys used).
```

This proves that the application is using the EC2 instance role to access S3.

---

# 📊 Application Verification Summary

| Component          | Verification           | Status |
| ------------------ | ---------------------- | ------ |
| Private ECR        | Repository created     | ✅      |
| ECR Authentication | Docker login           | ✅      |
| ECR `v2`           | Image pushed           | ✅      |
| ECR `latest`       | Image pushed           | ✅      |
| ECR Pull           | Image pulled           | ✅      |
| Django             | Container running      | ✅      |
| Nginx              | Container running      | ✅      |
| HTTPS              | Domain accessible      | ✅      |
| RDS                | Database connection    | ✅      |
| ElastiCache        | Redis cache test       | ✅      |
| S3                 | Write/read test        | ✅      |
| IAM Role           | S3 access through role | ✅      |
| Static Files       | Nginx serving CSS      | ✅      |

---

# 🐳 Current Docker Environment

Final verified Docker environment:

## Containers

```text
techcake-django
techcake-nginx
```

## Images

Required images:

```text
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:v2
nginx:alpine
```

The temporary image:

```text
alpine:latest
```

was used only for correcting Docker volume permissions and can be removed when no longer needed.

Remove it with:

```bash
docker image rm alpine:latest
```

---

# 💾 Docker Volumes

Current required volume:

```text
techcake-django-app-v2_static_volume
```

Check:

```bash
docker volume ls
```

Do not remove the active static volume unless the static-file architecture is changed.

---

# 🌐 Docker Networks

Application network:

```text
techcake-network
```

Check:

```bash
docker network ls
```

Default Docker networks:

```text
bridge
host
none
```

These are normal Docker networks and should not normally be removed.

---

# 🧹 Docker Cleanup

Check all containers:

```bash
docker ps -a
```

Check images:

```bash
docker images
```

Check volumes:

```bash
docker volume ls
```

Check networks:

```bash
docker network ls
```

Check disk usage:

```bash
docker system df
```

---

# 🧹 Remove Unused Resources

Only remove resources that are confirmed to be unused.

Remove unused containers:

```bash
docker container prune
```

Remove unused images:

```bash
docker image prune -a
```

Remove unused volumes:

```bash
docker volume prune
```

Remove unused networks:

```bash
docker network prune
```

### ⚠️ Warning

Do not blindly run cleanup commands on a production server.

Always inspect the resource first.

---

# 📁 Important Project Paths

| Purpose             | Path                                                              |
| ------------------- | ----------------------------------------------------------------- |
| Project             | `/home/djangov2/techcake-django-app-v2/`                          |
| Environment file    | `/home/djangov2/.env`                                             |
| Docker Compose      | `/home/djangov2/techcake-django-app-v2/docker-compose.yml`        |
| Dockerfile          | `/home/djangov2/techcake-django-app-v2/Dockerfile`                |
| Nginx configuration | `/home/djangov2/techcake-django-app-v2/nginx/`                    |
| Django settings     | `/home/djangov2/techcake-django-app-v2/techcake_site/settings.py` |
| Git ignore          | `/home/djangov2/techcake-django-app-v2/.gitignore`                |

---

# 🔧 Important Configuration

## Docker Compose

The Django service uses:

```yaml
image: 084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django:v2
```

Environment variables:

```yaml
env_file:
  - /home/djangov2/.env
```

Django static files:

```yaml
volumes:
  - static_volume:/app/staticfiles
```

Nginx mounts the same volume:

```yaml
volumes:
  - static_volume:/static:ro
```

This allows:

```text
Django → write static files
Nginx → read static files
```

---

# 🎨 Django Static Files

Django configuration:

```text
STATIC_URL=/static/
STATIC_ROOT=/app/staticfiles
```

Collect static files:

```bash
docker compose exec django \
python manage.py collectstatic --noinput
```

Verify:

```bash
docker exec -it techcake-django \
ls /app/staticfiles/admin/css
```

Expected files include:

```text
base.css
forms.css
login.css
dashboard.css
nav_sidebar.css
responsive.css
```

---

# 🌐 Nginx Static Files

Nginx serves static files directly:

```nginx
location /static/ {
    alias /static/;
    expires 30d;
    access_log off;
}
```

Test:

```bash
curl -I \
https://djangov2.shyamdev.nixlabs.in/static/admin/css/base.css
```

Expected:

```text
HTTP/2 200
content-type: text/css
```

---

# 🐛 Static File Issue Encountered

After pulling the application image from ECR, the Django Admin initially loaded without CSS.

Initial test:

```text
/static/admin/css/base.css
→ HTTP 400
```

Investigation showed:

```text
DEBUG=False
STATIC_URL=/static/
STATIC_ROOT=/app/staticfiles
```

The container did not have collected static files.

Running:

```bash
docker exec -it techcake-django \
python manage.py collectstatic --noinput
```

initially resulted in:

```text
PermissionError: [Errno 13] Permission denied:
'/app/staticfiles/admin'
```

---

# 🔧 Static Volume Permission Fix

The Docker volume was owned by root while the Django container runs as:

```text
djangov2
UID 1000
```

The volume ownership was corrected with:

```bash
docker run --rm \
-v techcake-django-app-v2_static_volume:/static \
alpine \
sh -c "chown -R 1000:1000 /static"
```

Then:

```bash
docker exec -it techcake-django \
python manage.py collectstatic --noinput
```

Successfully produced:

```text
125 static files copied to '/app/staticfiles'.
```

Nginx then successfully saw:

```text
/static/admin/css/
```

and the CSS test returned:

```text
HTTP/2 200
```

---

# ⚠️ Important Static File Note

The static volume was introduced to solve the static-file serving problem in the current two-container Django + Nginx architecture.

The original Task 6 requirements are focused on **ECR, clean pull-and-run, RDS, ElastiCache, and S3**.

The static volume is therefore a deployment implementation detail rather than an explicit Task 6 requirement.

---

# 🔍 Useful Docker Commands

## Check containers

```bash
docker ps
```

## Check all containers

```bash
docker ps -a
```

## Check images

```bash
docker images
```

## Check volumes

```bash
docker volume ls
```

## Check networks

```bash
docker network ls
```

## Check Docker disk usage

```bash
docker system df
```

## Check Django logs

```bash
docker logs techcake-django --tail 50
```

## Check Nginx logs

```bash
docker logs techcake-nginx --tail 50
```

## Follow Django logs

```bash
docker logs -f techcake-django
```

## Follow Nginx logs

```bash
docker logs -f techcake-nginx
```

---

# 🔍 Inspect the Django Container

```bash
docker inspect techcake-django
```

Image only:

```bash
docker inspect techcake-django \
--format '{{.Config.Image}}'
```

Environment:

```bash
docker inspect techcake-django \
--format '{{json .Config.Env}}'
```

> Avoid taking screenshots of environment output if it could expose sensitive values.

---

# 🔍 Inspect Nginx

List Nginx configuration:

```bash
docker exec -it techcake-nginx \
find /etc/nginx -type f
```

Test Nginx configuration:

```bash
docker exec techcake-nginx nginx -t
```

Reload:

```bash
docker exec techcake-nginx nginx -s reload
```

---

# 🔐 Security Checklist

Before committing or publishing the repository:

* [x] `.env` excluded using `.gitignore`
* [x] AWS credentials not hardcoded
* [x] Database password not stored in `docker-compose.yml`
* [x] Django secret loaded from environment
* [x] S3 access uses EC2 IAM role
* [x] Production secrets stored outside project
* [ ] Ensure `.env.template` contains only placeholders
* [ ] Never publish signed S3 URLs containing temporary credentials
* [ ] Never publish AWS session tokens

---

# 📝 `.gitignore`

Current important exclusions:

```gitignore
# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/
env/

# Django
staticfiles/
*.sqlite3
*.log

# Local env / secrets
.env
```

Optional additions:

```gitignore
# IDE
.vscode/
.idea/

# macOS
.DS_Store
```

---

# 📸 Task 6 Submission Evidence

The task requires proof of:

> ECR repository showing tags, push output, and pull-and-run on a clean host/state with RDS + ElastiCache + S3 working.

Recommended evidence:

## Screenshot 1 — ECR Repository

AWS Console:

```text
ECR
→ Repositories
→ shyamdev-techcake-django
→ Images
```

Show:

```text
latest
v2
```

Also show:

* Repository name
* Image digest
* Push time
* Last pulled time

---

## Screenshot 2 — ECR Push

Show successful commands:

```bash
docker push ...:v2
docker push ...:latest
```

Capture the successful digest output:

```text
v2: digest: sha256:...
latest: digest: sha256:...
```

---

## Screenshot 3 — Clean Pull and Run

Show:

```bash
docker compose down
```

then:

```bash
docker rmi ...
```

then:

```bash
docker compose pull
docker compose up -d
docker ps
```

This demonstrates that the application image was retrieved from ECR.

---

## Screenshot 4 — Managed Services Verification

Show:

```text
RDS Connected
```

Redis:

```text
ok
```

S3:

```text
S3 CHECK PASSED
```

This demonstrates:

```text
ECR
 │
 ▼
Django Container
 │
 ├── RDS       ✅
 ├── Redis     ✅
 └── S3        ✅
```

---

# 🧪 Final Verification Commands

Run these from:

```text
/home/djangov2/techcake-django-app-v2/
```

## 1. Compose validation

```bash
docker compose config
```

---

## 2. Container status

```bash
docker compose ps
```

---

## 3. Image verification

```bash
docker inspect techcake-django \
--format '{{.Config.Image}}'
```

---

## 4. Application test

```bash
curl -I https://djangov2.shyamdev.nixlabs.in
```

---

## 5. Static file test

```bash
curl -I \
https://djangov2.shyamdev.nixlabs.in/static/admin/css/base.css
```

---

## 6. RDS test

```bash
docker compose exec django \
python manage.py shell
```

```python
from django.db import connection
connection.ensure_connection()
print("RDS Connected")
exit()
```

---

## 7. Redis test

```bash
docker compose exec django \
python manage.py shell
```

```python
from django.core.cache import cache
cache.set("test", "ok")
print(cache.get("test"))
exit()
```

Expected:

```text
ok
```

---

## 8. S3 test

```bash
docker compose exec django \
python manage.py s3check
```

Expected:

```text
S3 CHECK PASSED
```

---

# 📋 Final Task 6 Checklist

* [x] Private ECR repository created
* [x] Docker authenticated with ECR
* [x] Image tagged `v2`
* [x] Image tagged `latest`
* [x] `v2` pushed to ECR
* [x] `latest` pushed to ECR
* [x] ECR repository verified
* [x] Local ECR image removed for clean-state testing
* [x] Image pulled from ECR
* [x] Container recreated from ECR image
* [x] Django application running
* [x] Nginx running
* [x] HTTPS working
* [x] RDS connectivity verified
* [x] ElastiCache/Redis connectivity verified
* [x] S3 read/write verified
* [x] S3 access confirmed through EC2 instance role
* [x] Versioned tagging strategy understood
* [x] Static files verified
* [x] Docker resources reviewed
* [x] No unnecessary Docker containers
* [x] No unnecessary application networks
* [x] Only required Docker volume retained

---

# 🏁 Conclusion

Task 6 successfully demonstrates the transition from a locally available Docker image to a **private Amazon ECR-based deployment workflow**.

The final deployment uses:

```text
Private ECR
    ↓
Docker Pull
    ↓
Django Container
    ↓
Nginx
    ↓
HTTPS
```

The Django application was successfully validated against:

```text
Amazon RDS          ✅
Amazon ElastiCache  ✅
Amazon S3           ✅
EC2 IAM Role        ✅
Amazon ECR          ✅
```

The use of both:

```text
v2
latest
```

provides a versioned release reference while retaining the convenience of the `latest` tag.

For production deployments, the **versioned tag should be preferred** because it provides a predictable and reproducible image reference.

---

# 📚 Quick Reference

### Project

```text
/home/djangov2/techcake-django-app-v2/
```

### Environment

```text
/home/djangov2/.env
```

### ECR

```text
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-techcake-django
```

### Version

```text
v2
```

### Latest

```text
latest
```

### Django Container

```text
techcake-django
```

### Nginx Container

```text
techcake-nginx
```

### Docker Network

```text
techcake-network
```

### Static Volume

```text
techcake-django-app-v2_static_volume
```

### Domain

```text
https://djangov2.shyamdev.nixlabs.in
```

### S3 Bucket

```text
shyamdev-dev-s3
```

### EC2 IAM Role

```text
shyamdev-dev-ec2-role
```

---

## ⭐ Key Lessons

1. **ECR is a private registry for Docker images.**
2. **Docker must authenticate with ECR before pulling/pushing private images.**
3. **Versioned tags such as `v2` make deployments reproducible.**
4. **`latest` alone is not a reliable production deployment reference.**
5. **A clean-state deployment proves that the application does not depend on the old local image.**
6. **EC2 IAM roles are preferred over hardcoded AWS credentials.**
7. **S3 access can be validated directly from inside the Django container.**
8. **RDS and ElastiCache remain external managed services; the Docker image only contains the application.**
9. **Nginx handles external HTTP/HTTPS traffic while Django runs internally on port `8000`.**
10. **Static files need an explicit production serving strategy when `DEBUG=False`.**
11. **Docker volumes should only be removed after confirming they are not required by the running deployment.**
12. **Always verify the actual image reference used by the running container when testing an ECR deployment.**

---

**Task 6 Status: ✅ COMPLETED**

```
```
