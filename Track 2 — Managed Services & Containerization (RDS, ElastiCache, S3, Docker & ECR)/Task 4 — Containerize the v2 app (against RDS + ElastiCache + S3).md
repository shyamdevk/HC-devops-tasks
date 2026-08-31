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
