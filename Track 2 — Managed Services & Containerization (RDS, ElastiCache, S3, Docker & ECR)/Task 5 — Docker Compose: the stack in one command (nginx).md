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
