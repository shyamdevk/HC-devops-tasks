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
