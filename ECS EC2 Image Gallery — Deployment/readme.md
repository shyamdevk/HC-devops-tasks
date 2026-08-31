# 🚀 ECS EC2 Image Gallery — Complete Deployment Guide

> **AWS ECS EC2 + ECR + ALB + ASG + Route 53 + ACM**
>
> A complete step-by-step reference for deploying a containerized Image Gallery application on **Amazon ECS using EC2 capacity**, exposing it through an **Application Load Balancer**, and securing it with **HTTPS**.

---

## 📌 Table of Contents

* [1. Project Overview](#1-project-overview)
* [2. Final Architecture](#2-final-architecture)
* [3. Prerequisites](#3-prerequisites)
* [4. VPC and Networking](#4-vpc-and-networking)
* [5. IAM Roles](#5-iam-roles)
* [6. ECR Repository](#6-ecr-repository)
* [7. ECS Cluster](#7-ecs-cluster)
* [8. ECS EC2 Instance and ASG](#8-ecs-ec2-instance-and-asg)
* [9. ECS Capacity Provider](#9-ecs-capacity-provider)
* [10. ECS Task Definition](#10-ecs-task-definition)
* [11. Target Group](#11-target-group)
* [12. Application Load Balancer](#12-application-load-balancer)
* [13. ECS Service](#13-ecs-service)
* [14. Route 53 DNS](#14-route-53-dns)
* [15. ACM Certificate](#15-acm-certificate)
* [16. HTTPS Listener](#16-https-listener)
* [17. Security Groups](#17-security-groups)
* [18. Final Verification](#18-final-verification)
* [19. Temporary Shutdown](#19-temporary-shutdown)
* [20. Starting the Environment Again](#20-starting-the-environment-again)
* [21. Troubleshooting](#21-troubleshooting)
* [22. Important Lessons](#22-important-lessons)
* [23. Final Deployment Checklist](#23-final-deployment-checklist)

---

# 1. Project Overview

The objective was to deploy the **Image Gallery application** using AWS managed services and containerization.

The application was deployed using:

| AWS Service                   | Purpose                                 |
| ----------------------------- | --------------------------------------- |
| **Amazon ECR**                | Stores the container image              |
| **Amazon ECS**                | Runs and manages the container          |
| **Amazon EC2**                | Provides compute capacity for ECS       |
| **Auto Scaling Group**        | Manages ECS EC2 capacity                |
| **ECS Capacity Provider**     | Connects ECS with the ASG               |
| **Application Load Balancer** | Receives and routes application traffic |
| **Target Group**              | Routes ALB traffic to ECS task IP       |
| **Route 53**                  | Provides custom DNS                     |
| **ACM**                       | Provides SSL/TLS certificate            |
| **IAM**                       | Provides required permissions           |
| **Security Groups**           | Controls network access                 |

Final application URL:

```text
https://image.gallery.shyamdev.nixlabs.in/
```

---

# 2. Final Architecture

The deployment works using the following flow:

```text
User
 │
 │ HTTPS :443
 ▼
Application Load Balancer
 │
 │ Target Group
 ▼
ECS Task Private IP
 │
 │ Container :5000
 ▼
Image Gallery Application
```

ECS capacity is provided separately:

```text
ECS Service
    │
    ▼
ECS Capacity Provider
    │
    ▼
Auto Scaling Group
    │
    ▼
EC2 Instance
    │
    ▼
ECS Container Instance
```

The complete request flow is:

```text
Browser
   ↓
Route 53
   ↓
ALB
   ↓
HTTPS :443
   ↓
Target Group
   ↓
ECS Task Private IP
   ↓
Container :5000
   ↓
Image Gallery
```

---

# 3. Prerequisites

Before starting, make sure you have:

* AWS account
* AWS CLI configured
* Docker installed
* Git installed
* GitHub repository
* Application source code
* AWS permissions for:

  * ECS
  * EC2
  * ECR
  * IAM
  * ALB
  * Route 53
  * ACM
  * Auto Scaling

Check AWS CLI:

```bash
aws --version
```

Check AWS identity:

```bash
aws sts get-caller-identity
```

Set the region:

```bash
aws configure
```

This deployment used:

```text
Region: us-east-1
```

---

# 4. VPC and Networking

The deployment used the existing VPC:

```text
VPC CIDR:
172.31.0.0/16
```

The ECS task uses VPC networking through:

```text
awsvpc
```

With `awsvpc`, every ECS task receives its own:

* Elastic Network Interface
* Private IPv4 address
* Network identity

Example:

```text
ECS Task
   ↓
Task ENI
   ↓
172.31.x.x
```

This is important because the Target Group uses:

```text
Target Type: IP addresses
```

## Availability Zones

The ALB was configured across multiple Availability Zones.

The deployment eventually required:

```text
us-east-1a
us-east-1b
us-east-1c
us-east-1d
```

### Important lesson

The ALB must be enabled in the Availability Zone where ECS places the task.

If ECS places a task in an AZ not enabled on the ALB, the Target Group can show:

```text
Target.NotInUse
```

with:

```text
Target is in an Availability Zone that is not enabled for the load balancer
```

This caused a temporary `503` during the deployment.

The solution was to add the required `us-east-1d` subnet to the ALB.

---

# 5. IAM Roles

## 5.1 ECS EC2 Instance Role

Configured:

```text
ecsInstanceRole
```

Required policies verified:

```text
AmazonEC2ContainerServiceforEC2Role
AmazonSSMManagedInstanceCore
```

This role allows the ECS EC2 instance to communicate with ECS and use SSM functionality.

---

## 5.2 ECS Task Execution Role

Configured:

```text
ecsTaskExecutionRole
```

This role is used by ECS to perform task-level operations such as pulling the container image from ECR.

---

# 6. ECR Repository

Created the ECR repository:

```text
shyamdev-image-gallery-app
```

Repository URI:

```text
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-image-gallery-app
```

## Build the Docker Image

From the application directory:

```bash
docker build -t shyamdev-image-gallery-app:v1 .
```

Check the image:

```bash
docker images
```

---

## Authenticate Docker with ECR

```bash
aws ecr get-login-password \
  --region us-east-1 \
  | docker login \
  --username AWS \
  --password-stdin \
  084828581506.dkr.ecr.us-east-1.amazonaws.com
```

---

## Tag the Image

```bash
docker tag \
  shyamdev-image-gallery-app:v1 \
  084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-image-gallery-app:v1
```

---

## Push the Image

```bash
docker push \
  084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-image-gallery-app:v1
```

Verify:

```bash
aws ecr describe-images \
  --repository-name shyamdev-image-gallery-app \
  --region us-east-1
```

---

## Immutable Image

The ECS task definition ultimately used the image digest:

```text
sha256:693d9053044b1f5b08a3c8092947669bec14c1c38df5ba998939191962dd5e3a
```

Using the digest ensures that ECS runs the exact image version rather than relying only on a mutable tag.

---

# 7. ECS Cluster

Created the ECS cluster:

```text
shyamdev-dev-image-galley-cluster
```

> Note: The cluster name contains `galley` as configured during the deployment.

The cluster uses:

```text
Amazon ECS
Launch Type: EC2
```

The EC2 instance acts as ECS compute capacity.

---

# 8. ECS EC2 Instance and ASG

Created an ECS EC2 instance using an EC2 launch template.

Example instance:

```text
Instance ID:
i-00ed8fea82c388f63
```

The instance is registered with ECS as a container instance.

Expected ECS state:

```text
Status: ACTIVE
Running Tasks: 1
Pending Tasks: 0
```

---

## Auto Scaling Group

The ECS EC2 instance is managed by an Auto Scaling Group.

Configured capacity:

```text
Minimum: 0
Desired: 1
Maximum: 1
```

When the environment is being used:

```text
Desired = 1
```

When the environment is not being used:

```text
Desired = 0
```

This is important for reducing unnecessary EC2 usage/cost.

---

# 9. ECS Capacity Provider

The ECS cluster/service uses an **ECS Capacity Provider** connected to the Auto Scaling Group.

The Capacity Provider allows ECS to manage EC2 capacity based on the ECS service's task requirements.

Basic relationship:

```text
ECS Service
     ↓
Capacity Provider
     ↓
Auto Scaling Group
     ↓
EC2
```

### Important lesson

If:

```text
ECS Desired Tasks = 1
```

and:

```text
ASG Desired Capacity = 0
```

ECS Capacity Provider managed scaling may launch an EC2 instance again because ECS still needs capacity for the task.

Therefore, when shutting down the environment completely:

```text
ECS Desired Tasks = 0
ASG Desired Capacity = 0
```

---

# 10. ECS Task Definition

Created task definition:

```text
shyamdev-image-gallery-task
```

The Task Definition acts as the blueprint for running the application container.

## Task Configuration

```text
Launch Type: EC2
CPU: 256
Memory: 512 MiB
Network Mode: awsvpc
Operating System: Linux
Architecture: X86_64
```

## Container Configuration

```text
Container Name: image-gallery
Container Port: 5000
Protocol: HTTP
Essential: Yes
```

Container image:

```text
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-image-gallery-app
```

The image was pinned using its SHA256 digest.

---

## Why `awsvpc`?

`awsvpc` gives each ECS task its own network interface and private IP.

Example:

```text
ECS Task
   ↓
ENI
   ↓
172.31.x.x
```

This allows the ALB to directly communicate with the ECS task through an IP-based Target Group.

---

# 11. Target Group

Created:

```text
shyamdev-image-gallery-tg
```

Configuration:

```text
Target Type: IP addresses
Protocol: HTTP
Port: 5000
IP Address Type: IPv4
Health Check Path: /
Health Check Port: Traffic Port
```

## Why IP Target Type?

Because the ECS task uses:

```text
awsvpc
```

The task gets its own private IP.

Therefore:

```text
ALB
 ↓
Target Group
 ↓
ECS Task Private IP
 ↓
Container :5000
```

The Target Group does **not** manually register the EC2 instance.

ECS automatically registers the task's private IP when the ECS service starts the task.

Example:

```text
172.31.81.10:5000
```

When the task stops, ECS automatically deregisters it.

---

## Target Health

The target was verified as:

```text
State: healthy
```

This confirms that the ALB can successfully reach the application on port `5000`.

---

# 12. Application Load Balancer

Created:

```text
shyamdev-image-gallery-alb
```

Configuration:

```text
Type: Application Load Balancer
Scheme: Internet-facing
IP Address Type: IPv4
```

The ALB was configured across multiple Availability Zones.

---

## ALB DNS

The ALB provides an AWS DNS name similar to:

```text
shyamdev-image-gallery-alb-58187111.us-east-1.elb.amazonaws.com
```

The ALB DNS was tested before configuring the custom domain.

---

# 13. ECS Service

Created:

```text
shyamdev-image-gallery-task-service
```

The ECS service uses:

```text
Task Definition:
shyamdev-image-gallery-task
```

Desired task count:

```text
1
```

Normal healthy state:

```text
Desired: 1
Running: 1
Pending: 0
```

The ECS service connects the container to:

```text
shyamdev-image-gallery-tg
```

with:

```text
Container:
image-gallery

Port:
5000
```

---

## ECS Service + Target Group

The service automatically registers the task with the Target Group.

When a new task starts:

```text
ECS Task starts
      ↓
Task gets private IP
      ↓
ECS registers IP
      ↓
ALB health check
      ↓
Healthy
      ↓
Traffic allowed
```

When a task becomes unhealthy:

```text
Health check fails
      ↓
ECS replaces task
      ↓
New task starts
      ↓
New IP registered
      ↓
Health check
      ↓
Healthy
```

---

# 14. Route 53 DNS

Hosted Zone:

```text
shyamdev.nixlabs.in
```

Created DNS record:

```text
image.gallery.shyamdev.nixlabs.in
```

The DNS record points to:

```text
Application Load Balancer
```

DNS verification:

```bash
nslookup image.gallery.shyamdev.nixlabs.in
```

The hostname successfully resolved to the ALB.

---

# 15. ACM Certificate

Created an ACM public certificate for:

```text
image.gallery.shyamdev.nixlabs.in
```

Certificate ARN:

```text
arn:aws:acm:us-east-1:084828581506:certificate/ae004acf-de7c-4746-8ba1-2bda2c6cbeb4
```

DNS validation was completed using Route 53.

Certificate status:

```text
ISSUED
```

The certificate was attached to the ALB HTTPS listener.

---

# 16. HTTPS Listener

Configured two ALB listeners.

## HTTP :80

```text
HTTP :80
    ↓
301 Redirect
    ↓
HTTPS :443
```

This ensures normal HTTP traffic is redirected to HTTPS.

---

## HTTPS :443

```text
HTTPS :443
    ↓
ACM Certificate
    ↓
Target Group
    ↓
ECS Task :5000
```

The ACM certificate was configured as the default certificate for the HTTPS listener.

---

# 17. Security Groups

Two security groups were used.

## 17.1 ALB Security Group

Example:

```text
sg-03164f838b0dad102
```

Inbound:

```text
HTTP  :80  → 0.0.0.0/0
HTTPS :443 → 0.0.0.0/0
```

The ALB accepts public web traffic.

---

## 17.2 ECS/EC2 Security Group

Example:

```text
sg-02a2e63894ceae8cb
```

Application access:

```text
TCP 5000
Source: ALB Security Group
```

SSH access:

```text
TCP 22
Source: Administrator IP /32
```

### Important Security Rule

Do **not** expose:

```text
TCP 5000 → 0.0.0.0/0
```

The application should be reachable through the ALB rather than directly from the Internet.

The final intended flow is:

```text
Internet
   ↓
ALB :443
   ↓
ECS SG :5000
   ↓
Container
```

---

# 18. Final Verification

After deployment, verify every major component.

---

## 18.1 Check ECS Tasks

```bash
aws ecs list-tasks \
  --cluster shyamdev-dev-image-galley-cluster \
  --service-name shyamdev-image-gallery-task-service \
  --region us-east-1
```

Expected:

```text
1 task
```

---

## 18.2 Check ECS Service

```bash
aws ecs describe-services \
  --cluster shyamdev-dev-image-galley-cluster \
  --services shyamdev-image-gallery-task-service \
  --region us-east-1 \
  --query 'services[0].{Desired:desiredCount,Running:runningCount,Pending:pendingCount}'
```

Expected:

```text
Desired: 1
Running: 1
Pending: 0
```

---

## 18.3 Check Target Health

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:084828581506:targetgroup/shyamdev-image-gallery-tg/b6abf88a7a48fa08 \
  --region us-east-1 \
  --query 'TargetHealthDescriptions[].{IP:Target.Id,Port:Target.Port,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
  --output table
```

Expected:

```text
State: healthy
```

---

## 18.4 Check ALB Listeners

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn $(aws elbv2 describe-load-balancers \
  --names shyamdev-image-gallery-alb \
  --region us-east-1 \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text) \
  --region us-east-1
```

Verify:

```text
HTTP :80
HTTPS :443
```

---

## 18.5 Check DNS

```bash
nslookup image.gallery.shyamdev.nixlabs.in
```

Expected:

```text
image.gallery.shyamdev.nixlabs.in
```

resolves to the ALB.

---

## 18.6 Test HTTP

```bash
curl -I http://image.gallery.shyamdev.nixlabs.in/
```

Expected:

```text
HTTP/1.1 301
```

with redirect to HTTPS.

---

## 18.7 Test HTTPS

```bash
curl -I https://image.gallery.shyamdev.nixlabs.in/
```

Expected:

```text
HTTP/1.1 200 OK
```

---

## 18.8 Browser Test

Open:

```text
https://image.gallery.shyamdev.nixlabs.in/
```

Expected:

```text
Image Gallery
```

The browser should show the HTTPS lock icon.

---

# 19. Temporary Shutdown

For this lab environment, the team workflow is to reduce capacity when the environment is not being used.

Because ECS Capacity Provider can automatically increase ASG capacity when ECS still requires tasks, **do not only change the ASG desired count while leaving the ECS service desired count at `1`.**

## Step 1 — Stop ECS Tasks

Change:

```text
ECS Service Desired Tasks
1 → 0
```

Wait until:

```text
Running Tasks = 0
```

---

## Step 2 — Set ASG Desired Capacity to 0

Change:

```text
ASG Desired Capacity
1 → 0
```

Keep:

```text
Minimum = 0
Maximum = 1
```

Final shutdown state:

```text
ECS Desired Tasks = 0
ASG Desired Capacity = 0
EC2 Instances = 0
```

This prevents the Capacity Provider from immediately launching another instance to satisfy an ECS task requirement.

### Important

The website will be temporarily unavailable while there is no ECS task/EC2 capacity.

The following resources remain configured:

* ECS cluster
* ECS service
* Task definition
* ECR
* ALB
* Target Group
* Route 53
* ACM
* Security Groups
* ASG
* Launch Template

---

# 20. Starting the Environment Again

When the environment is needed again:

## Step 1 — Start EC2 Capacity

Change ASG:

```text
Desired Capacity:
0 → 1
```

Keep:

```text
Minimum: 0
Maximum: 1
```

The ASG launches an EC2 instance using the existing launch template.

---

## Step 2 — Wait for ECS Registration

The EC2 instance should become:

```text
Running
```

and then register with ECS:

```text
ACTIVE
```

---

## Step 3 — Start ECS Task

Change the ECS service:

```text
Desired Tasks:
0 → 1
```

ECS then places the task on the available EC2 capacity.

---

## Step 4 — Wait for Target Health

Wait until:

```text
ECS Task: RUNNING
Target Group: healthy
```

---

## Step 5 — Test Website

Open:

```text
https://image.gallery.shyamdev.nixlabs.in/
```

The Image Gallery should load again.

---

# 21. Troubleshooting

## 21.1 Website returns 503

A `503` from the ALB commonly means that there is currently no usable healthy target.

Check:

```bash
aws elbv2 describe-target-health \
  --target-group-arn YOUR_TARGET_GROUP_ARN \
  --region us-east-1
```

Look for:

```text
State
Reason
Description
```

---

## 21.2 Target shows `Target.Timeout`

Example:

```text
State: unhealthy
Reason: Target.Timeout
Description: Request timed out
```

Check:

* ECS task is running
* Container is listening on port `5000`
* ECS security group allows port `5000` from ALB SG
* Target Group port is `5000`
* Health check path is `/`

Test from inside the VPC if appropriate:

```bash
curl http://TASK_PRIVATE_IP:5000/
```

A successful response should return:

```text
HTTP/1.1 200 OK
```

---

## 21.3 Target shows `Target.NotInUse`

Example:

```text
Target is in an Availability Zone that is not enabled for the load balancer
```

This means:

```text
ECS Task AZ
      ≠
ALB Enabled AZ
```

Fix by enabling the ECS task's Availability Zone/subnet on the ALB.

---

## 21.4 ECS keeps replacing tasks

Check service events:

```bash
aws ecs describe-services \
  --cluster shyamdev-dev-image-galley-cluster \
  --services shyamdev-image-gallery-task-service \
  --region us-east-1 \
  --query 'services[0].events[0:10].[createdAt,message]' \
  --output table
```

Look for messages such as:

```text
Amazon ECS replaced 1 tasks due to an unhealthy status.
```

Then investigate Target Group health.

---

## 21.5 ECS shows 1 Running + 1 Provisioning

This can temporarily happen while ECS replaces an unhealthy task.

During replacement:

```text
Old Task → Running
New Task → Provisioning
```

After the new task becomes healthy:

```text
Old Task → Stopped
New Task → Running
```

Final expected state:

```text
Running: 1
Provisioning/Pending: 0
```

Do not manually terminate tasks during a normal replacement unless troubleshooting requires it.

---

## 21.6 ASG automatically returns Desired Capacity to 1

If:

```text
ECS Desired Tasks = 1
```

and:

```text
ASG Desired Capacity = 0
```

the ECS Capacity Provider can request EC2 capacity again.

Correct shutdown procedure:

```text
ECS Desired Tasks → 0
        ↓
Wait for tasks to stop
        ↓
ASG Desired Capacity → 0
```

---

# 22. Important Lessons

## ECS Task Definition

A Task Definition is the **blueprint** for running a container.

It defines:

* Image
* CPU
* Memory
* Network mode
* Ports
* Roles
* Container configuration

---

## ECS Service

The ECS Service maintains the required number of tasks.

For this deployment:

```text
Desired Tasks = 1
```

---

## `awsvpc`

`awsvpc` gives every ECS task its own network identity.

```text
Task
 ↓
ENI
 ↓
Private IP
```

---

## Target Group

The Target Group does not directly use the EC2 instance in this deployment.

Because we use:

```text
awsvpc
+
Target Type: IP
```

ECS registers:

```text
Task Private IP :5000
```

Example:

```text
172.31.81.10:5000
```

---

## ALB

The ALB provides the public entry point.

It handles:

```text
HTTP :80
HTTPS :443
```

and forwards requests to the Target Group.

---

## ASG

The ASG manages ECS EC2 capacity.

It controls how many EC2 instances are available for ECS.

---

## Capacity Provider

The ECS Capacity Provider connects:

```text
ECS
 ↓
ASG
 ↓
EC2
```

It can automatically request EC2 capacity when ECS needs to place tasks.

---

## Route 53

Route 53 provides the human-friendly application hostname:

```text
image.gallery.shyamdev.nixlabs.in
```

instead of requiring users to use the ALB DNS name.

---

## ACM

ACM provides the SSL/TLS certificate used by the ALB.

---

# 23. Final Deployment Checklist

Before final submission, verify all of the following.

### EC2

```text
[ ] Instance running when environment is active
[ ] ECS container instance = ACTIVE
[ ] Correct IAM instance role
```

### ASG

```text
[ ] ASG exists
[ ] Min = 0
[ ] Desired = 1 when active
[ ] Max = 1
[ ] Instance = InService
```

### ECR

```text
[ ] Repository exists
[ ] Image pushed
[ ] v1 tag exists
[ ] Image digest verified
```

### ECS

```text
[ ] Cluster = ACTIVE
[ ] Task Definition exists
[ ] awsvpc configured
[ ] Container port = 5000
[ ] ECS Service = ACTIVE
[ ] Desired = 1
[ ] Running = 1
[ ] Pending = 0
```

### Target Group

```text
[ ] Target type = IP
[ ] Protocol = HTTP
[ ] Port = 5000
[ ] Health check path = /
[ ] Target = healthy
```

### ALB

```text
[ ] ALB = Active
[ ] Internet-facing
[ ] Required AZs enabled
[ ] HTTP :80 configured
[ ] HTTPS :443 configured
```

### Security Groups

```text
[ ] ALB :80 open as required
[ ] ALB :443 open as required
[ ] ECS :5000 accepts traffic only from ALB SG
[ ] SSH restricted to required administrator IP
[ ] No public :5000 access
```

### Route 53

```text
[ ] image.gallery.shyamdev.nixlabs.in exists
[ ] Record points to ALB
[ ] DNS resolves correctly
```

### ACM

```text
[ ] Certificate exists
[ ] Certificate = ISSUED
[ ] Correct domain
[ ] Attached to HTTPS listener
```

### Final Application

```text
[ ] HTTP redirects to HTTPS
[ ] HTTPS returns 200
[ ] Browser loads Image Gallery
[ ] HTTPS certificate is valid
[ ] Target Group remains healthy
```

---

# 🎯 Final Result

The Image Gallery application was successfully deployed using an **ECS EC2-based architecture**.

The final deployment provides:

```text
ECR
 ↓
ECS Task Definition
 ↓
ECS Service
 ↓
ECS EC2 Capacity
 ↓
Target Group
 ↓
Application Load Balancer
 ↓
HTTPS / ACM
 ↓
Route 53
 ↓
image.gallery.shyamdev.nixlabs.in
```

Final application URL:

```text
https://image.gallery.shyamdev.nixlabs.in/
```

Final expected active state:

```text
ECS Service       → ACTIVE
Desired Tasks     → 1
Running Tasks     → 1
Pending Tasks     → 0
Target Health     → HEALTHY
ALB               → ACTIVE
HTTPS             → ENABLED
DNS               → CONFIGURED
Application       → ACCESSIBLE
```

> **Deployment completed successfully with ECS on EC2, ECR-based image delivery, ASG-managed capacity, ALB routing, IP-based ECS targets, Route 53 DNS, and ACM-secured HTTPS.**
