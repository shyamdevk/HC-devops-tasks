# Image Gallery App — AWS ECS Deployment

This document records the complete deployment of the **Image Gallery Application** on AWS using **Amazon ECS with the EC2 launch type**.

The application uses Amazon RDS for the database, Amazon S3 for image storage, AWS Secrets Manager and SSM Parameter Store for secure configuration, Application Load Balancer for application access, ECS Service Auto Scaling for scalability, and AWS CodePipeline with CodeBuild for CI/CD.

**GitHub Repository:**
[https://github.com/shyamdevk/Image-Gallery-App-New](https://github.com/shyamdevk/Image-Gallery-App-New)

**Live Application:**
[https://image.gallery.shyamdev.nixlabs.in/](https://image.gallery.shyamdev.nixlabs.in/)

---

# 1. Task Requirements

The task required:

* Deploy the application on Amazon ECS using EC2 launch type.
* Configure Amazon RDS as the backend database.
* Secure database credentials and application secrets.
* Configure Amazon S3 for image storage.
* Configure ECS Service Auto Scaling.
* Implement CI/CD using AWS CodePipeline.
* Provide deployment documentation and presentation.

---

# 2. Application Preparation

Before deploying to ECS, the application was prepared as a Dockerized application.

## Docker Configuration

The application was configured to run using Gunicorn.

```text
Application Port: 5000
Deployment Mode: aws
Debug Mode: False
Container: image-gallery
```

The application was tested locally inside a Docker container on the EC2 instance used for Docker and ECR operations.

### Verification

```text
Container: image-gallery-test
Server: Gunicorn
Listening: 0.0.0.0:5000
Health Test: HTTP/1.1 200 OK
```

The application was confirmed to run successfully in the Docker environment. 

---

# 3. Create VPC Networking

The ECS infrastructure and RDS database were configured inside the same VPC.

## VPC

```text
VPC ID:
vpc-0b47e6e84369dad77
```

The required subnets, route configuration, security groups, NACLs, and DNS settings were configured for communication between the application and database.

## Security Groups

```text
RDS Security Group:
sg-0b9cf7efbed7a5fd2

Application/ECS EC2 Security Group:
sg-065b2bac1563a64af
```

RDS inbound access was configured for TCP port `3306` from the application/ECS security group.

## Verification

* ECS infrastructure and RDS are inside the same VPC.
* Local VPC routing is available.
* RDS is private.
* RDS uses port `3306`.
* Required NACL traffic is allowed.
* VPC DNS Support is enabled.
* VPC DNS Hostnames are enabled.

These networking configurations were verified during deployment. 

---

# 4. Create Amazon RDS

Amazon RDS MySQL was created as the backend database for the application.

## RDS Configuration

```text
Instance:
shyamdev-dev-image-gallery-rds02

Database:
image_gallery

Database User:
imagegallery

Port:
3306
```

The RDS instance was configured as a private database within the VPC.

## Verification

* RDS instance status is `available`.
* Application successfully connects to RDS.
* Database operations were successfully tested from the application. 

---

# 5. Create Amazon S3

Amazon S3 was configured for storing application images.

## S3 Configuration

```text
Bucket:
shyamdev-dev-image-gallery

Deployment Mode:
DEPLOYMENT_MODE=aws
```

The bucket was configured with restricted public access.

The application uses IAM-based access to S3 and generates pre-signed URLs for private image access.

## Verification

* Images were successfully uploaded to S3.
* Uploaded images were stored in the bucket.
* Images were successfully viewed.
* Images were successfully downloaded.
* S3 access is provided through IAM permissions. 

---

# 6. Create Secure Configuration

Sensitive configuration was separated from the application source code.

Two AWS-managed services were used for secure configuration.

## AWS Secrets Manager

The RDS password was configured through AWS Secrets Manager.

The ECS task definition references the secret and injects the required value into the application container.

```text
AWS Secrets Manager
        ↓
ECS Task Definition
        ↓
Application Container
        ↓
RDS
```

## SSM Parameter Store

Application parameters were created under:

```text
/IMAGE-GALLERY-DEV/
```

Parameters included:

```text
RDS_DATABASE
RDS_ENDPOINT
RDS_USER
RDS_PASSWORD
SECRET_KEY
```

Sensitive parameters were stored as `SecureString`.

The final configuration used:

```text
RDS Password → AWS Secrets Manager
SECRET_KEY   → SSM Parameter Store
```

## Verification

* Sensitive credentials are not hard-coded in the application.
* ECS is configured to retrieve the required secure values.
* Secret values were not exposed in the repository or screenshots. 

---

# 7. Create Amazon ECR

Amazon ECR was created to store the Docker image used by ECS.

## ECR Configuration

```text
Repository:
shyamdev-dev-image-gallery

Image:
image-gallery-app:production
```

Repository URI:

```text
084828581506.dkr.ecr.us-east-1.amazonaws.com/shyamdev-dev-image-gallery
```

## Build and Push

The Docker image was built and pushed to ECR.

## Verification

* Docker image build completed successfully.
* Image was tagged successfully.
* Image was pushed successfully to ECR.
* ECS uses the image stored in ECR. 

---

# 8. Create ECS Cluster

An ECS cluster was created using the **EC2 launch type**.

## ECS Cluster

```text
Cluster:
shyamdev-dev-image-gallery-cluster
```

EC2 capacity was configured for the ECS cluster using an ECS Capacity Provider backed by EC2 Auto Scaling infrastructure.

## Verification

* ECS cluster is active.
* EC2 container capacity is registered.
* ECS service can launch application tasks successfully. 

---

# 9. Create ECS Task Definition

A task definition was created for the application.

## Final Task Definition

```text
Task Definition:
shyamdev-dev-image-gallery:13

CPU:
512

Memory:
768 MB

Container:
image-gallery

Container Port:
5000
```

The task definition was configured with:

* ECR image
* Application environment configuration
* Secure parameter/secret references
* CloudWatch logging
* Container port `5000`

## CloudWatch Log Group

```text
/ecs/shyamdev-dev-image-gallery
```

The ECS container uses the `awslogs` logging driver.

## Verification

* Final task definition revision was successfully registered.
* ECS service was updated to use revision `13`.
* Container started successfully.
* CloudWatch logging was configured. 

---

# 10. Create Target Group

An Application Load Balancer target group was created for the ECS application.

## Target Configuration

```text
Application Port:
5000
```

## Health Check

```text
Protocol:
HTTP

Path:
/health

Port:
Traffic Port

Success Code:
200
```

The `/health` endpoint was added to the application for ALB health checks.

## Verification

* ECS target was registered with the target group.
* Health check returned HTTP `200`.
* ECS target was verified as `Healthy`. 

---

# 11. Create Application Load Balancer

An Application Load Balancer was configured to provide external access to the ECS application.

## Listener Configuration

```text
HTTPS :443 → Target Group
```

HTTP traffic was redirected:

```text
HTTP :80 → HTTPS :443
```

The unnecessary HTTP port `5000` listener was removed.

An ACM certificate was configured for HTTPS access.

## Verification

* ALB successfully forwards traffic to ECS.
* ECS target is healthy.
* HTTPS access was verified.
* Domain access was verified.
* ACM certificate was verified. 

---

# 12. Create ECS Service

An ECS service was created to maintain the application task.

## ECS Service

```text
Service:
shyamdev-dev-image-gallery-service-01
```

The service was configured with:

```text
Launch Type:
EC2

Network Mode:
awsvpc

Container Port:
5000

Desired Tasks:
1
```

The service uses the final task definition revision.

## Verification

* ECS service is running.
* Application task is in the `RUNNING` state.
* Service uses the ECR application image.
* ALB is connected to the service. 

---

# 13. Configure ECS Service Auto Scaling

ECS Service Auto Scaling was configured to automatically adjust the number of application tasks.

## Scaling Configuration

```text
Minimum Tasks:
1

Maximum Tasks:
3

CPU Target:
60%
```

Scaling Policy:

```text
image-gallery-cpu-scaling
```

## Verification

* Scaling policy is active.
* Minimum task count is configured as `1`.
* Maximum task count is configured as `3`.
* CPU target tracking is configured at `60%`.
* ECS can increase or decrease the number of running tasks based on demand. 

---

# 14. Create AWS CodeBuild Project

AWS CodeBuild was configured to build the application Docker image.

## Build Process

```text
GitHub Source
     ↓
CodeBuild
     ↓
Docker Build
     ↓
ECR Push
     ↓
imagedefinitions.json
```

CodeBuild performs the Docker image build and pushes the resulting image to ECR.

## Verification

* CodeBuild project completed successfully.
* Docker image was built.
* Image was pushed to ECR.
* `imagedefinitions.json` was generated successfully. 

---

# 15. Create AWS CodePipeline

AWS CodePipeline was configured to automate application deployment.

## Pipeline

```text
GitHub → CodePipeline → CodeBuild → ECR → ECS
```

The GitHub `main` branch was configured as the source.

A new push to the repository automatically triggers the pipeline.

## Deployment Process

1. Code is pushed to GitHub.
2. CodePipeline detects the change.
3. CodeBuild builds the Docker image.
4. The image is pushed to ECR.
5. CodePipeline deploys the latest image to ECS.
6. ECS starts the updated application task.
7. ALB performs the application health check.

## Verification

* GitHub source integration works.
* Pipeline detects new changes.
* CodeBuild completes successfully.
* New Docker image is pushed to ECR.
* ECS service is updated automatically.
* Latest pipeline execution completed successfully. 

---

# 16. Final Application Verification

The complete deployment was verified through the live application.

## Live URL

[https://image.gallery.shyamdev.nixlabs.in/](https://image.gallery.shyamdev.nixlabs.in/)

## Verified

* Application is accessible through HTTPS.
* ALB routes traffic to the ECS task.
* ECS task is running.
* Target group reports the ECS target as healthy.
* Application connects successfully to RDS.
* Images are stored in S3.
* Images can be viewed and downloaded.
* Secure configuration is retrieved through AWS-managed services.
* ECS Service Auto Scaling is active.
* GitHub changes can trigger the CI/CD pipeline.
* Latest application version was successfully deployed through CodePipeline.

---

# 17. Final AWS Resources

| Service              | Resource                                |
| -------------------- | --------------------------------------- |
| ECS Cluster          | `shyamdev-dev-image-gallery-cluster`    |
| ECS Service          | `shyamdev-dev-image-gallery-service-01` |
| ECS Task Definition  | `shyamdev-dev-image-gallery:13`         |
| ECR Repository       | `shyamdev-dev-image-gallery`            |
| RDS Instance         | `shyamdev-dev-image-gallery-rds02`      |
| RDS Database         | `image_gallery`                         |
| S3 Bucket            | `shyamdev-dev-image-gallery`            |
| SSM Parameter Path   | `/IMAGE-GALLERY-DEV/`                   |
| CloudWatch Log Group | `/ecs/shyamdev-dev-image-gallery`       |
| Scaling Policy       | `image-gallery-cpu-scaling`             |
| Application Port     | `5000`                                  |
| HTTPS Port           | `443`                                   |

---

# Final Status

The Image Gallery application was successfully deployed and verified on Amazon ECS using the EC2 launch type.

The complete implementation includes:

* Amazon ECS
* EC2 Capacity Provider
* Amazon ECR
* Application Load Balancer
* Amazon RDS MySQL
* Amazon S3
* AWS Secrets Manager
* SSM Parameter Store
* CloudWatch Logs
* ECS Service Auto Scaling
* AWS CodeBuild
* AWS CodePipeline

**Deployment Status: COMPLETED**

**Verification Status: PASSED**

This version is much better for your GitHub repo because someone can **rebuild the environment later by following the sections in order**, while the final verification/evidence at the bottom proves what was actually completed. The sequence also follows the actual work recorded in the ticket rather than inventing a different deployment history. 
