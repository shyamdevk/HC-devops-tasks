# 🖼️ Image Gallery Application — AWS Deployment & CI/CD

> Complete deployment documentation for the Image Gallery Flask application.
>
> This README is written as a future reference to remember how the entire project was created, configured, secured, deployed, and verified.

---

# 📌 1. Project Overview

The Image Gallery application is deployed on an AWS EC2 instance using Docker Compose.

The application uses:

- Amazon EC2 — Application server
- Docker — Application containerization
- Docker Compose — Container management
- Amazon RDS MySQL — Production database
- Amazon S3 — Image/object storage
- IAM Role — AWS access from EC2
- GitHub OIDC — Secure GitHub-to-AWS authentication
- GitHub Actions — CI/CD pipeline
- AWS Systems Manager (SSM) — Remote deployment to EC2
- HTTPS — Public application access
- Dedicated Linux users — Security and isolation

---

# 🏗️ 2. Final Architecture

```text
GitHub Repository
       │
       │ Push to main
       ▼
GitHub Actions
       │
       │ OIDC
       ▼
AWS IAM Role
       │
       ▼
AWS Systems Manager (SSM)
       │
       ▼
EC2
       │
       └── Docker Compose
              │
              └── Flask + Gunicorn
                     │
                     ├── Amazon RDS MySQL
                     │
                     └── Amazon S3
````

---

# ☁️ 3. AWS Region

The project was deployed in:

```text
us-east-1
```

---

# 🖥️ 4. EC2 Instance

Application server:

```text
Instance ID:
i-01e58f800895ff030
```

Application directory:

```text
/opt/imagegallery/Image-Galley-App
```

Dedicated Linux user:

```text
imagegallery
```

Home directory:

```text
/home/imagegallery
```

---

# 👤 5. Create Dedicated Linux User

The application should not be operated directly as root.

Create the user:

```bash
sudo useradd -m -s /bin/bash imagegallery
```

Verify:

```bash
getent passwd imagegallery
```

Expected:

```text
imagegallery:x:1001:1001::/home/imagegallery:/bin/bash
```

Verify home directory:

```bash
sudo ls -ld /home/imagegallery
```

Expected ownership:

```text
imagegallery:imagegallery
```

---

# 📂 6. Create Application Directory

Create the application parent directory:

```bash
sudo mkdir -p /opt/imagegallery
```

Set ownership:

```bash
sudo chown imagegallery:imagegallery /opt/imagegallery
```

Application location:

```text
/opt/imagegallery/Image-Galley-App
```

Set ownership:

```bash
sudo chown -R imagegallery:imagegallery /opt/imagegallery
```

Verify:

```bash
sudo ls -ld /opt/imagegallery
sudo ls -ld /opt/imagegallery/Image-Galley-App
```

---

# 📦 7. Get the Application Code

Switch to the application user:

```bash
sudo -iu imagegallery
```

Clone the repository:

```bash
git clone https://github.com/shyamdevk/Image-Galley-App.git /opt/imagegallery/Image-Galley-App
```

Enter the application directory:

```bash
cd /opt/imagegallery/Image-Galley-App
```

Verify:

```bash
pwd
git status
```

Expected location:

```text
/opt/imagegallery/Image-Galley-App
```

---

# 🗄️ 8. Create Amazon RDS

Create an Amazon RDS MySQL database.

Database details used by the application:

```text
Database:
image_gallery

Port:
3306
```

RDS instance:

```text
shyamdev-dev-image-gallery-rds01
```

RDS endpoint:

```text
shyamdev-dev-image-gallery-rds01.c830oeksoq9i.us-east-1.rds.amazonaws.com
```

---

# 👤 9. Create Dedicated RDS Application User

The application should use a dedicated database user instead of the RDS administrative user.

Application database user:

```text
imagegallery_app
```

Connect using an administrative RDS account and create the user:

```sql
CREATE USER 'imagegallery_app'@'%' IDENTIFIED BY 'STRONG_PASSWORD';
```

Grant application permissions:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX
ON image_gallery.* TO 'imagegallery_app'@'%';
```

Verify:

```sql
SHOW GRANTS FOR 'imagegallery_app'@'%';
```

Expected permissions include:

```text
SELECT
INSERT
UPDATE
DELETE
CREATE
INDEX
ALTER
```

---

# 🧪 10. Verify RDS Database

Connect to RDS:

```bash
mysql -h RDS_ENDPOINT \
-P 3306 \
-u imagegallery_app \
-p
```

Use the database:

```sql
USE image_gallery;
```

Verify tables:

```sql
SHOW TABLES;
```

Expected:

```text
images
users
```

---

# 🪣 11. Create Amazon S3 Bucket

Create/configure the S3 bucket used by the application.

Bucket:

```text
shyamdev-dev-image-gallery-bucket
```

The application uses S3 for image/object storage.

---

# 🔐 12. Configure EC2 IAM Role

The EC2 instance uses an IAM role instead of storing AWS access keys inside the application.

This allows the application to access AWS services using temporary credentials.

The EC2 role used in the project:

```text
shyamdev-dev-image-gallery
```

S3 access is provided through IAM permissions.

Do NOT store:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside the application `.env`.

---

# 👤 13. Configure EC2 → SSM

AWS Systems Manager is used by GitHub Actions to execute deployment commands on the EC2 instance.

The EC2 instance must:

* Be registered as an SSM managed instance
* Have the required SSM permissions through its IAM role
* Have the SSM Agent available/running
* Have network access required by SSM

Verify the instance is available through Systems Manager before using GitHub Actions deployment.

---

# 🔑 14. Generate Flask SECRET_KEY

Generate a secure secret:

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

This produces a 64-character hexadecimal value.

Example:

```text
<GENERATED_SECRET_KEY>
```

Do not put the real value into GitHub, README, screenshots, or source code.

---

# 🔒 15. Production `.env`

Production secrets are stored outside the Git repository.

Location:

```text
/home/imagegallery/.env
```

Create:

```bash
sudo -u imagegallery nano /home/imagegallery/.env
```

The file contains the required production configuration:

```text
DEPLOYMENT_MODE=aws
FLASK_APP=run.py
FLASK_ENV=production
FLASK_PORT=5000
SECRET_KEY=<generated-secret>
DEBUG=False
MAX_FILE_SIZE=16777216
ALLOWED_EXTENSIONS=jpg,jpeg,png,gif,webp
UPLOAD_FOLDER=app/static/uploads

RDS_ENDPOINT=<RDS-ENDPOINT>
RDS_DATABASE=image_gallery
RDS_USER=imagegallery_app
RDS_PASSWORD=<RDS-PASSWORD>
RDS_PORT=3306

S3_BUCKET_NAME=shyamdev-dev-image-gallery-bucket
AWS_REGION=us-east-1
```

---

# 🔐 16. Secure `.env` Permissions

Set ownership:

```bash
sudo chown imagegallery:imagegallery /home/imagegallery/.env
```

Set permissions:

```bash
sudo chmod 600 /home/imagegallery/.env
```

Verify:

```bash
sudo ls -l /home/imagegallery/.env
```

Expected:

```text
-rw------- ... imagegallery imagegallery ... /home/imagegallery/.env
```

The `.env` file must never be committed to Git.

---

# 🚫 17. Protect `.env` Using `.gitignore`

`.gitignore` contains:

```gitignore
.env
.env.local
```

Verify:

```bash
git ls-files .env
```

Expected:

```text
(no output)
```

---

# ⚙️ 18. Configure Flask Environment Loading

The application explicitly loads the production environment file.

In:

```text
app/__init__.py
```

use:

```python
from dotenv import load_dotenv

load_dotenv('/home/imagegallery/.env')
```

The Flask secret key is loaded from the environment:

```python
app.config['SECRET_KEY'] = os.environ['SECRET_KEY']
```

This prevents the application from silently using a weak development fallback.

---

# 🗄️ 19. Configure RDS in Flask

The application reads:

```text
RDS_ENDPOINT
RDS_DATABASE
RDS_USER
RDS_PASSWORD
RDS_PORT
```

from `.env`.

The database connection is constructed from environment variables.

No database password should be hardcoded into Python source files.

---

# 🪣 20. Configure S3 in Flask

The application reads:

```text
S3_BUCKET_NAME
AWS_REGION
```

from the environment.

AWS credentials are NOT stored in the application configuration.

The EC2 IAM role provides AWS access.

---

# 🐳 21. Dockerfile

The application uses:

```dockerfile
FROM python:3.10-slim
```

Dependencies are installed from:

```text
requirements.txt
```

The application runs under a dedicated container user.

Example:

```dockerfile
RUN mkdir -p app/static/uploads && \
    useradd --create-home --shell /bin/bash appuser && \
    chown -R appuser:appuser /app

USER appuser
```

Gunicorn is used to run Flask:

```dockerfile
CMD ["gunicorn", "--workers", "3", "--bind", "0.0.0.0:5000", "app:app"]
```

---

# 👤 22. Docker Non-Root User

After building the image, verify:

```bash
sudo docker exec image_gallery_app whoami
```

Expected:

```text
appuser
```

Verify UID/GID:

```bash
sudo docker exec image_gallery_app id
```

Expected:

```text
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```

This confirms that the application does not run as root inside the container.

---

# 🐳 23. Production Docker Compose

Production uses:

```text
docker-compose.aws.yml
```

The environment file is loaded from:

```yaml
env_file:
  - /home/imagegallery/.env
```

The application port is bound to localhost:

```yaml
ports:
  - "127.0.0.1:5000:5000"
```

This prevents direct public access to port 5000.

---

# 🧪 24. Validate Docker Compose

Run:

```bash
sudo docker compose -f docker-compose.aws.yml config
```

Check that:

* Correct image/build configuration is present
* Correct RDS endpoint is configured
* Correct S3 bucket is configured
* `DEBUG=False`
* `DEPLOYMENT_MODE=aws`
* `.env` is being loaded from `/home/imagegallery/.env`
* Port 5000 is bound to `127.0.0.1`

IMPORTANT:

Do not share screenshots containing real:

```text
RDS_PASSWORD
SECRET_KEY
```

---

# 🔨 25. Build Application

From:

```text
/opt/imagegallery/Image-Galley-App
```

run:

```bash
sudo docker compose -f docker-compose.aws.yml build
```

Expected:

```text
Successfully built
```

---

# 🚀 26. Start Application

Run:

```bash
sudo docker compose -f docker-compose.aws.yml up -d
```

Verify:

```bash
sudo docker ps
```

Expected:

```text
image_gallery_app
Up
127.0.0.1:5000->5000/tcp
```

---

# 📋 27. Verify Gunicorn

Check logs:

```bash
sudo docker logs --tail 50 image_gallery_app
```

Expected messages include:

```text
Starting gunicorn
Listening at: http://0.0.0.0:5000
Booting worker
```

---

# 🗄️ 28. Verify Application → RDS

Run from the application container:

```bash
sudo docker exec image_gallery_app python3 -c "
from app import create_app, db
app = create_app()
with app.app_context():
    print('DATABASE:', db.engine.url.database)
    print('HOST:', db.engine.url.host)
    print('DB CONNECTION:', db.engine.connect().closed is False)
"
```

Expected:

```text
DATABASE: image_gallery
HOST: <RDS-ENDPOINT>
DB CONNECTION: True
```

---

# 🪣 29. Verify Application → S3

Run:

```bash
sudo docker exec image_gallery_app python3 -c "
from app import create_app
from app.utils.s3 import check_s3_bucket_access
import os

app = create_app()

with app.app_context():
    bucket = os.getenv('S3_BUCKET_NAME')
    print('S3 BUCKET:', bucket)
    print('S3 ACCESS:', check_s3_bucket_access(bucket))
"
```

Expected:

```text
S3 BUCKET: shyamdev-dev-image-gallery-bucket
S3 ACCESS: True
```

---

# 🌐 30. Security Groups

## EC2 Security Group

Required inbound traffic:

| Type  | Port | Source                  |
| ----- | ---: | ----------------------- |
| SSH   |   22 | Administrator public IP |
| HTTP  |   80 | Internet                |
| HTTPS |  443 | Internet                |

Do NOT expose:

```text
5000
```

to the Internet.

---

## RDS Security Group

RDS MySQL:

```text
Port 3306
```

Source:

```text
EC2 Security Group
```

Do NOT use:

```text
0.0.0.0/0
```

for RDS port 3306.

---

# 🌐 31. DNS

Application hostname:

```text
image.gallery1.shyamdev.nixlabs.in
```

The DNS record points to the current EC2 public IP in this setup.

IMPORTANT:

An EC2 public IP can change after a stop/start if an Elastic IP is not attached.

Therefore:

```text
EC2 stop/start
        ↓
Possible public IP change
        ↓
DNS record may need updating
```

This does NOT affect GitHub Actions deployment because the deployment uses the EC2 Instance ID through SSM.

---

# 🔐 32. GitHub OIDC Setup

GitHub Actions authenticates to AWS using OIDC.

This avoids storing long-term AWS access keys in GitHub.

Create/configure the AWS IAM OIDC provider for:

```text
token.actions.githubusercontent.com
```

The GitHub Actions workflow requests an OIDC token.

AWS IAM verifies the GitHub identity using the role trust policy.

---

# 👤 33. GitHub Actions IAM Role

IAM role used:

```text
shyamdev-dev-image-gallery-github-actions
```

Role ARN:

```text
arn:aws:iam::084828581506:role/shyamdev-dev-image-gallery-github-actions
```

The role is trusted by GitHub Actions through OIDC.

The trust policy should restrict access to the intended GitHub repository/branch.

Repository:

```text
shyamdevk/Image-Galley-App
```

Branch:

```text
main
```

---

# 🔐 34. GitHub Actions IAM Permissions

The GitHub Actions role needs only the AWS permissions required by the workflow.

The deployment workflow needs to:

```text
Send SSM command
Wait for command execution
Read SSM command output
```

Relevant SSM permissions include the required:

```text
ssm:SendCommand
ssm:GetCommandInvocation
```

Additional permissions should be granted only when required.

Use least privilege whenever possible.

---

# 📁 35. Create GitHub Actions Workflow

Create:

```text
.github/workflows/deploy.yml
```

The workflow runs when code is pushed to:

```text
main
```

---

# 🚀 36. Final GitHub Actions Workflow

The actual deployment workflow uses:

```yaml
name: Deploy Image Gallery to AWS

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    name: Deploy to EC2
    runs-on: ubuntu-latest

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::084828581506:role/shyamdev-dev-image-gallery-github-actions
          aws-region: us-east-1

      - name: Deploy application through SSM
        shell: bash
        run: |
          COMMAND_ID=$(aws ssm send-command \
            --instance-ids i-01e58f800895ff030 \
            --document-name "AWS-RunShellScript" \
            --parameters 'commands=[
              "cd /opt/imagegallery/Image-Galley-App",
              "sudo -u imagegallery git pull --ff-only origin main",
              "sudo docker compose -f docker-compose.aws.yml build",
              "sudo docker compose -f docker-compose.aws.yml up -d",
              "sudo docker ps",
              "curl -f http://127.0.0.1:5000"
            ]' \
            --region us-east-1 \
            --query 'Command.CommandId' \
            --output text)

          echo "SSM Command ID: $COMMAND_ID"

          aws ssm wait command-executed \
            --command-id "$COMMAND_ID" \
            --instance-id i-01e58f800895ff030 \
            --region us-east-1

          aws ssm get-command-invocation \
            --command-id "$COMMAND_ID" \
            --instance-id i-01e58f800895ff030 \
            --region us-east-1 \
            --query '{Status:Status,Output:StandardOutputContent,Error:StandardErrorContent}' \
            --output json
```

---

# 🔄 37. Understand the GitHub Actions Workflow

The important part is:

```text
GitHub push
     ↓
GitHub Actions starts
     ↓
GitHub OIDC token
     ↓
AWS IAM role
     ↓
SSM SendCommand
     ↓
EC2 Instance ID
     ↓
git pull
     ↓
Docker build
     ↓
Docker Compose up
     ↓
Local curl health check
```

---

# 🆔 38. Important: Deployment Uses EC2 Instance ID

The workflow uses:

```text
i-01e58f800895ff030
```

It does NOT use:

```text
EC2 public IP
EC2 public DNS
Route 53 hostname
Elastic IP
```

Therefore, changing the EC2 public IP does not prevent the GitHub Actions deployment.

SSM identifies the target using the EC2 Instance ID.

---

# 🌐 39. Public DNS vs CI/CD

These are separate:

## CI/CD

```text
GitHub
 ↓
GitHub Actions
 ↓
AWS OIDC
 ↓
IAM
 ↓
SSM
 ↓
EC2 Instance ID
```

Public DNS is not involved.

## Website

```text
Browser
 ↓
image.gallery1.shyamdev.nixlabs.in
 ↓
EC2 public IP
 ↓
Application
```

Therefore:

```text
DNS failure ≠ CI/CD failure
```

---

# 🧪 40. First CI/CD Test

After creating the workflow, commit and push:

```bash
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Actions deployment"
git push origin main
```

Open:

```text
GitHub → Repository → Actions
```

Find:

```text
Deploy Image Gallery to AWS
```

Verify:

```text
Configure AWS credentials       ✅
Deploy application through SSM  ✅
```

Overall:

```text
Success ✅
```

---

# 🧪 41. Test Automatic Deployment

To prove CI/CD works, make a harmless README change.

For example, add:

```md
## CI/CD Verification

This change verifies automatic deployment through GitHub Actions and AWS SSM.
```

Then:

```bash
git add README.md
git commit -m "Test CI/CD deployment"
git push origin main
```

---

# 🔍 42. Verify GitHub Actions

Open:

```text
GitHub → Actions
```

The workflow should automatically start.

Expected:

```text
Deploy to EC2
        ✅
```

The workflow should:

```text
Authenticate to AWS
        ↓
Send SSM command
        ↓
Pull latest Git code
        ↓
Build Docker image
        ↓
Restart container
        ↓
Run health check
```

---

# 🖥️ 43. Verify New Commit on EC2

After GitHub Actions succeeds:

```bash
sudo -u imagegallery git log -1 --oneline
```

The latest commit should match the latest GitHub `main` commit.

Verify:

```bash
sudo -u imagegallery git status --short
```

Expected:

```text
(no output)
```

---

# 🐳 44. Verify Container After CI/CD

Run:

```bash
sudo docker ps
```

Expected:

```text
image_gallery_app
Up
```

Verify the container user:

```bash
sudo docker exec image_gallery_app whoami
```

Expected:

```text
appuser
```

Verify:

```bash
sudo docker exec image_gallery_app id
```

Expected:

```text
uid=1000(appuser) gid=1000(appuser)
```

---

# 🌐 45. Verify Local Application

Run:

```bash
curl -I http://127.0.0.1:5000
```

Expected:

```text
HTTP/1.1 200 OK
```

---

# 🔒 46. Verify Public HTTPS

Run:

```bash
curl -I https://image.gallery1.shyamdev.nixlabs.in
```

Expected:

```text
HTTP/1.1 200 OK
```

The application should also load successfully in a browser.

---

# 🔐 47. Final `.env` Verification

Verify:

```bash
sudo ls -ld /home/imagegallery
sudo ls -l /home/imagegallery/.env
```

Expected:

```text
/home/imagegallery
    owner: imagegallery
    permissions: 700

/home/imagegallery/.env
    owner: imagegallery
    permissions: 600
```

Never display the actual `.env` contents.

---

# 🔍 48. Secret Leak Check

Check tracked files:

```bash
sudo -u imagegallery git ls-files .env
```

Expected:

```text
(no output)
```

Search the Git repository for common credential patterns:

```bash
sudo -u imagegallery git grep -nEi \
'(AKIA[0-9A-Z]{16}|AWS_SECRET_ACCESS_KEY|AWS_ACCESS_KEY_ID|RDS_PASSWORD=|MYSQL_PASSWORD=|SECRET_KEY=)' \
-- ':!*.md' ':!*.example'
```

Expected:

```text
(no output)
```

Check hardcoded environment assignments:

```bash
sudo -u imagegallery git grep -nE \
'^[A-Z_]+=(.+)$' \
-- ':!*.md' ':!*.example'
```

Expected:

```text
(no output)
```

---

# 🧹 49. Repository Cleanup

The production environment file remains outside Git:

```text
/home/imagegallery/.env
```

Unnecessary environment files were removed from the repository:

```text
.env.aws
.env.example
```

Final configuration changes included:

```text
app/__init__.py
docker-compose.aws.yml
```

---

# 📝 50. Important Git Changes

Final hardening changes:

```text
load_dotenv('/home/imagegallery/.env')
```

instead of:

```text
load_dotenv()
```

And:

```python
app.config['SECRET_KEY'] = os.environ['SECRET_KEY']
```

instead of using a development fallback.

Docker Compose uses:

```yaml
env_file:
  - /home/imagegallery/.env
```

instead of:

```yaml
env_file:
  - .env
```

---

# 📌 51. Final Git Commit

The security hardening commit was:

```text
c025562 Harden production environment configuration
```

Verify:

```bash
sudo -u imagegallery git log -1 --oneline
```

Expected:

```text
c025562 Harden production environment configuration
```

Verify remote:

```bash
sudo -u imagegallery git log -2 --oneline
```

Expected:

```text
c025562 (HEAD -> main, origin/main, origin/HEAD)
Harden production environment configuration

8c1e399 Run container as non-root user
```

---

# 📸 52. Recommended Evidence Screenshots

Do not capture every command.

Recommended major evidence:

## Screenshot 1 — GitHub Actions

Show:

```text
Deploy Image Gallery to AWS
Success ✅
Deploy to EC2 ✅
```

---

## Screenshot 2 — Docker Container

Show:

```bash
sudo docker ps
```

with:

```text
image_gallery_app
Up
127.0.0.1:5000->5000/tcp
```

---

## Screenshot 3 — Non-Root Container

Show:

```bash
sudo docker exec image_gallery_app whoami
sudo docker exec image_gallery_app id
```

Expected:

```text
appuser
uid=1000(appuser) gid=1000(appuser)
```

---

## Screenshot 4 — RDS Connectivity

Show:

```text
DATABASE: image_gallery
HOST: RDS endpoint
DB CONNECTION: True
```

Do not show passwords.

---

## Screenshot 5 — S3 Connectivity

Show:

```text
S3 BUCKET: shyamdev-dev-image-gallery-bucket
S3 ACCESS: True
```

---

## Screenshot 6 — `.env` Security

Show:

```bash
sudo ls -l /home/imagegallery/.env
```

Showing:

```text
-rw------- imagegallery imagegallery
```

Do NOT show the `.env` contents.

---

## Screenshot 7 — EC2 Security Group

Show:

```text
SSH   22   Administrator IP
HTTP  80   0.0.0.0/0
HTTPS 443  0.0.0.0/0
```

---

## Screenshot 8 — RDS Security Group

Show:

```text
MySQL/Aurora
3306
Source: EC2 Security Group
```

---

## Screenshot 9 — HTTPS Application

Show the browser with:

```text
https://image.gallery1.shyamdev.nixlabs.in
```

or:

```bash
curl -I https://image.gallery1.shyamdev.nixlabs.in
```

with:

```text
HTTP/1.1 200 OK
```

---

# 🚫 53. Do Not Add Unnecessary Services

This project does NOT require:

```text
AWS CodePipeline
AWS CodeDeploy
Application Load Balancer
Auto Scaling Group
ECS
ECR
```

The implemented CI/CD solution is:

```text
GitHub Actions
+
AWS OIDC
+
IAM
+
SSM
+
EC2
+
Docker Compose
```

ASG/ALB are not required unless the project requirements specifically ask for high availability, load balancing, or automatic scaling.

---

# 🧠 54. Important Future Recall

## If I forget how deployment works:

Remember:

```text
GitHub
  ↓
Push to main
  ↓
GitHub Actions
  ↓
OIDC
  ↓
AWS IAM Role
  ↓
SSM
  ↓
EC2 Instance ID
  ↓
git pull
  ↓
Docker build
  ↓
docker compose up
  ↓
curl localhost:5000
```

---

## If the website is down:

Check:

```bash
sudo docker ps
```

Then:

```bash
sudo docker logs --tail 50 image_gallery_app
```

Then:

```bash
curl -I http://127.0.0.1:5000
```

Then:

```bash
curl -I https://image.gallery1.shyamdev.nixlabs.in
```

If localhost works but HTTPS does not, investigate:

```text
DNS
Security Group
HTTP/HTTPS configuration
Public IP
```

---

## If GitHub Actions fails:

Check:

```text
GitHub Actions
    ↓
OIDC
    ↓
IAM Role
    ↓
SSM
    ↓
EC2 SSM availability
    ↓
Instance ID
    ↓
Application directory
    ↓
Git pull
    ↓
Docker build
    ↓
Docker Compose
```

---

## If RDS connection fails:

Check:

```text
.env
    ↓
RDS_ENDPOINT
RDS_DATABASE
RDS_USER
RDS_PASSWORD
RDS_PORT
    ↓
RDS Security Group
    ↓
EC2 → RDS :3306
```

---

## If S3 access fails:

Check:

```text
S3_BUCKET_NAME
    ↓
EC2 IAM Role
    ↓
S3 permissions
    ↓
Bucket
```

Do not add AWS access keys to the application.

---

# 🏁 55. Final Verification Checklist

```text
[✓] EC2 created
[✓] Dedicated imagegallery Linux user
[✓] Application stored under /opt/imagegallery
[✓] Production .env stored under /home/imagegallery
[✓] .env permissions set to 600
[✓] .env owned by imagegallery
[✓] .env excluded from Git
[✓] RDS MySQL configured
[✓] Dedicated RDS application user
[✓] RDS database verified
[✓] RDS tables verified
[✓] S3 bucket configured
[✓] EC2 IAM role configured
[✓] S3 IAM access verified
[✓] Flask SECRET_KEY generated
[✓] SECRET_KEY loaded from environment
[✓] No weak SECRET_KEY fallback
[✓] Docker configured
[✓] Docker Compose configured
[✓] Container runs as appuser
[✓] Container not running as root
[✓] Port 5000 bound to localhost
[✓] Gunicorn running
[✓] RDS connection verified
[✓] S3 connection verified
[✓] Security Groups verified
[✓] GitHub OIDC configured
[✓] GitHub Actions IAM role configured
[✓] SSM deployment configured
[✓] GitHub Actions deployment successful
[✓] README CI/CD test successful
[✓] Latest Git commit reached EC2
[✓] Docker container restarted successfully
[✓] Local HTTP 200 verified
[✓] Public HTTPS 200 verified
```

---

# 🎯 56. Final Result

The Image Gallery application is successfully deployed on AWS.

The final deployment uses:

```text
EC2
+
Docker Compose
+
Gunicorn
+
Amazon RDS
+
Amazon S3
+
IAM
+
GitHub OIDC
+
GitHub Actions
+
AWS SSM
```

The application runs as a non-root Docker user and uses a dedicated Linux server user.

Production secrets are stored outside the Git repository in:

```text
/home/imagegallery/.env
```

The application successfully connects to:

```text
Amazon RDS
Amazon S3
```

GitHub Actions automatically deploys changes pushed to the `main` branch.

Final CI/CD verification confirmed:

```text
GitHub Push
    ↓
GitHub Actions
    ↓
AWS OIDC
    ↓
IAM
    ↓
SSM
    ↓
EC2
    ↓
Docker Build
    ↓
Docker Compose
    ↓
Application
```

Final application status:

```text
Docker       → Running       ✅
Non-root     → appuser       ✅
RDS          → Connected     ✅
S3           → Accessible    ✅
HTTPS        → HTTP 200      ✅
CI/CD        → Success       ✅
Security     → Verified      ✅
```

# 🟢 PROJECT STATUS: COMPLETED AND VERIFIED

---

# 📋 57. Quick Reference

| Item                   | Value                                        |
| ---------------------- | -------------------------------------------- |
| AWS Region             | `us-east-1`                                  |
| EC2 Instance           | `i-01e58f800895ff030`                        |
| Linux User             | `imagegallery`                               |
| Application Path       | `/opt/imagegallery/Image-Galley-App`         |
| `.env` Path            | `/home/imagegallery/.env`                    |
| Docker Container       | `image_gallery_app`                          |
| Container User         | `appuser`                                    |
| Application Port       | `5000`                                       |
| RDS Instance           | `shyamdev-dev-image-gallery-rds01`           |
| RDS Database           | `image_gallery`                              |
| RDS Port               | `3306`                                       |
| RDS App User           | `imagegallery_app`                           |
| S3 Bucket              | `shyamdev-dev-image-gallery-bucket`          |
| GitHub Repository      | `shyamdevk/Image-Galley-App`                 |
| GitHub Branch          | `main`                                       |
| GitHub Actions Role    | `shyamdev-dev-image-gallery-github-actions`  |
| Deployment Service     | AWS Systems Manager                          |
| Public URL             | `https://image.gallery1.shyamdev.nixlabs.in` |
| Final Hardening Commit | `c025562`                                    |

```

**This one is the version I'd keep in the GitHub repo.** It is specifically structured so that if you come back months later and forget *"How did I connect GitHub to EC2?"*, *"Why does changing the EC2 IP not break CI/CD?"*, *"Where is my `.env`?"*, *"How did RDS/S3 connect?"*, or *"How did I create the pipeline?"*, the answers are all in one place.
```
