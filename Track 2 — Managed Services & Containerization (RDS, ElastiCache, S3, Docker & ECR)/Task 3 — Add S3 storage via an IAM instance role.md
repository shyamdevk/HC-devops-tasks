# 🚀 Task 3 — Amazon S3 Integration with EC2 IAM Role

> **TechCake Django App v2 — Secure S3 Media Storage**

[![AWS](https://img.shields.io/badge/AWS-S3-orange?logo=amazon-aws)](https://aws.amazon.com/s3/)
[![IAM](https://img.shields.io/badge/AWS-IAM-blue?logo=amazon-aws)](https://aws.amazon.com/iam/)
[![Django](https://img.shields.io/badge/Django-S3%20Storage-092E20?logo=django)](https://www.djangoproject.com/)
[![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python)](https://www.python.org/)
[![Security](https://img.shields.io/badge/Security-IAM%20Role-success)](https://aws.amazon.com/iam/)

---

## 📌 Overview

This task integrates the **TechCake Django App v2** with **Amazon S3** for object/media storage.

The application runs on an **Amazon EC2 instance** and accesses the S3 bucket using an **IAM Instance Role**.

### 🔐 Important Security Principle

> **No AWS Access Keys are stored inside the application.**

Instead, the EC2 instance obtains temporary AWS credentials automatically through its attached IAM Instance Role.

### Authentication Flow

```text
                         AWS
                          │
                          ▼
                 ┌─────────────────┐
                 │   EC2 Instance  │
                 │                 │
                 │ shyamdev-dev-   │
                 │ web01           │
                 └────────┬────────┘
                          │
                          │ IAM Instance Role
                          ▼
                 ┌─────────────────┐
                 │ IAM Role        │
                 │                 │
                 │ shyamdev-dev-   │
                 │ ec2-role        │
                 └────────┬────────┘
                          │
                          │ S3 Permissions
                          ▼
                 ┌─────────────────┐
                 │ Amazon S3       │
                 │                 │
                 │ shyamdev-dev-s3 │
                 └─────────────────┘
```

---

# 🎯 Task Objectives

The main objectives of this task were:

- Create an Amazon S3 bucket.
- Keep the S3 bucket private.
- Enable Block Public Access.
- Use Bucket Owner Enforced object ownership.
- Create a least-privilege IAM policy.
- Attach the S3 policy to the EC2 IAM Role.
- Keep the existing SSM permissions.
- Configure Django to use Amazon S3.
- Avoid storing AWS Access Keys in the application.
- Verify IAM Role authentication.
- Verify successful S3 write and read operations.

---

# 🏗️ Architecture

```text
┌─────────────────────────────────────────────┐
│                  AWS Cloud                  │
│                                             │
│  ┌──────────────────────┐                   │
│  │      EC2 Instance    │                   │
│  │                      │                   │
│  │ shyamdev-dev-web01   │                   │
│  │                      │                   │
│  │ Django Application   │                   │
│  │ TechCake App v2      │                   │
│  └──────────┬───────────┘                   │
│             │                               │
│             │ IAM Instance Role             │
│             ▼                               │
│  ┌──────────────────────┐                   │
│  │      IAM Role        │                   │
│  │                      │                   │
│  │ shyamdev-dev-        │                   │
│  │ ec2-role             │                   │
│  └──────────┬───────────┘                   │
│             │                               │
│             │ Least-Privilege S3 Policy     │
│             ▼                               │
│  ┌──────────────────────┐                   │
│  │      Amazon S3       │                   │
│  │                      │                   │
│  │ shyamdev-dev-s3      │                   │
│  └──────────────────────┘                   │
│                                             │
└─────────────────────────────────────────────┘
```

---

# 📋 Environment Details

| Component | Configuration |
|---|---|
| Application | TechCake Django App v2 |
| EC2 Instance | `shyamdev-dev-web01` |
| EC2 Instance ID | `i-0a4864284196ba37b` |
| IAM Role | `shyamdev-dev-ec2-role` |
| S3 Bucket | `shyamdev-dev-s3` |
| AWS Region | `us-east-1` |
| S3 Policy | `shyamdev-dev-s3-policy` |
| SSM Policy | `AmazonSSMManagedInstanceCore` |
| Django Storage Backend | `S3Boto3Storage` |
| Python Environment | `venv` |

---

# 🪣 1. Create the S3 Bucket

Create the bucket from:

```text
AWS Console
    ↓
S3
    ↓
Create bucket
```

### Bucket Configuration

```text
Bucket Name:
shyamdev-dev-s3

Region:
us-east-1
```

### Security Configuration

The following configuration was used:

- ✅ General purpose bucket
- ✅ Bucket Owner Enforced
- ✅ ACLs disabled
- ✅ Block all public access
- ✅ Versioning disabled
- ✅ SSE-S3 default encryption

---

# 🔒 2. S3 Public Access Configuration

The bucket must remain private.

### Required Configuration

```text
Block all public access
        ↓
        ON
```

All four Block Public Access options remain enabled.

### Why?

The application does not require the S3 bucket to be publicly accessible.

The EC2 application accesses S3 using its IAM Role.

```text
Internet
   │
   │ ❌ No public S3 access
   ▼
S3 Bucket
   ▲
   │
   │ ✅ IAM authenticated access
   │
EC2 / Django Application
```

---

# 👤 3. IAM Role

The EC2 instance uses the following IAM Role:

```text
shyamdev-dev-ec2-role
```

The existing role was reused because the EC2 instance already required SSM access.

### Attached Policies

```text
shyamdev-dev-ec2-role
│
├── AmazonSSMManagedInstanceCore
│
└── shyamdev-dev-s3-policy
```

### Important

An EC2 instance uses **one attached IAM Instance Role**.

Therefore, instead of attaching a second role, the required S3 policy was added to the existing EC2 role.

This also preserves Session Manager / SSM functionality.

---

# 🛡️ 4. Least-Privilege S3 Policy

The custom policy is:

```text
shyamdev-dev-s3-policy
```

The policy grants only the permissions required by the application.

## Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BucketList",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::shyamdev-dev-s3"
    },
    {
      "Sid": "ObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::shyamdev-dev-s3/*"
    }
  ]
}
```

---

# 🔍 5. IAM Permissions Explained

The policy separates **bucket-level** and **object-level** permissions.

## `s3:ListBucket`

```text
Resource:
arn:aws:s3:::shyamdev-dev-s3
```

Allows the application to list objects in the bucket.

---

## `s3:GetObject`

```text
Resource:
arn:aws:s3:::shyamdev-dev-s3/*
```

Allows the application to read objects.

---

## `s3:PutObject`

```text
Resource:
arn:aws:s3:::shyamdev-dev-s3/*
```

Allows the application to upload/write objects.

---

## ❌ Permissions intentionally NOT granted

The application does not require broad administrative permissions such as:

```text
s3:DeleteBucket
s3:CreateBucket
s3:DeleteObject
s3:PutBucketPolicy
s3:PutBucketAcl
s3:* 
```

This follows the **Principle of Least Privilege**.

---

# 🔑 6. IAM Role Authentication

The EC2 instance obtains temporary credentials from the EC2 Instance Metadata Service (IMDS).

This was verified using IMDSv2.

### Request IMDSv2 Token

```bash
TOKEN=$(curl -X PUT \
"http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

### Retrieve IAM Role

```bash
curl \
-H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Expected:

```text
shyamdev-dev-ec2-role
```

---

# 🧪 7. Verify AWS Identity

Install/use AWS CLI and run:

```bash
aws sts get-caller-identity
```

Successful output confirmed:

```text
arn:aws:sts::084828581506:assumed-role/shyamdev-dev-ec2-role/i-0a4864284196ba37b
```

### What this proves

The EC2 instance is using:

```text
IAM Role
    ↓
shyamdev-dev-ec2-role
```

and not a permanent IAM User access key.

---

# 🔐 8. No AWS Access Keys in the Application

The application does **not** contain:

```env
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_SESSION_TOKEN=
```

Instead, only the S3 configuration is provided through environment variables.

Example:

```env
AWS_STORAGE_BUCKET_NAME=shyamdev-dev-s3
AWS_S3_REGION_NAME=us-east-1
```

---

# 🐍 9. Django S3 Configuration

The application uses:

```python
AWS_STORAGE_BUCKET_NAME = os.environ.get(
    "AWS_STORAGE_BUCKET_NAME",
    "",
)

AWS_S3_REGION_NAME = os.environ.get(
    "AWS_S3_REGION_NAME",
    "",
)

AWS_S3_FILE_OVERWRITE = False
AWS_DEFAULT_ACL = None
AWS_QUERYSTRING_AUTH = True
```

The default storage backend is:

```python
STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
    },
    "staticfiles": {
        "BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage",
    },
}
```

---

# 🔄 10. How Django Obtains AWS Credentials

No credentials are hard-coded in `settings.py`.

The application uses the standard boto3 credential provider chain.

On EC2, the flow is:

```text
Django
   │
   ▼
django-storages
   │
   ▼
boto3
   │
   ▼
EC2 Instance Metadata Service
   │
   ▼
Temporary IAM Role Credentials
   │
   ▼
shyamdev-dev-ec2-role
   │
   ▼
Amazon S3
```

This provides temporary credentials instead of permanent access keys.

---

# 📁 11. Project Location

The Django application is located at:

```text
/home/djangov2/techcake-django-app-v2
```

Enter the project:

```bash
cd /home/djangov2
cd techcake-django-app-v2
```

Activate the virtual environment:

```bash
source venv/bin/activate
```

Expected prompt:

```text
(venv)
```

---

# 🧰 12. S3 Check Management Command

The project contains the custom Django management command:

```text
./congrats/management/commands/s3check.py
```

Verify it exists:

```bash
find . -path "*/management/commands/*"
```

Expected relevant file:

```text
./congrats/management/commands/s3check.py
```

---

# ⚠️ Important: How to Run `s3check`

Do **NOT** execute the Python file directly:

```bash
./congrats/management/commands/s3check.py
```

This can produce:

```text
Permission denied
```

because the file is a Django management command and does not need to be executable.

### Correct command

Run:

```bash
python manage.py s3check
```

---

# ✅ 13. S3 Write + Read Verification

Run:

```bash
python manage.py s3check
```

Successful verification:

```text
Bucket: shyamdev-dev-s3
Writing test object: s3check/techcake-XXXXXXXX.txt ...
Reading it back ...
S3 CHECK PASSED — wrote and read back 60 bytes.
```

The final output confirms:

```text
Credentials were provided by the instance role (no access keys used).
```

---

# 🎯 What `s3check` Proves

The command verifies both directions:

```text
Django Application
       │
       │ PUT
       ▼
Amazon S3
       │
       │ GET
       ▼
Django Application
```

Therefore:

- ✅ S3 authentication works.
- ✅ `s3:PutObject` works.
- ✅ `s3:GetObject` works.
- ✅ IAM Role credentials work.
- ✅ Bucket configuration works.
- ✅ Django S3 configuration works.
- ✅ No AWS Access Keys are being used.

---

# 🗂️ 14. S3 Test Object

The successful test creates an object similar to:

```text
s3check/
└── techcake-bfd42f51.txt
```

The exact filename may change because the command generates a unique test filename.

---

# 🔧 15. Troubleshooting

## Problem 1 — `Permission denied` running `s3check.py`

### Incorrect

```bash
./congrats/management/commands/s3check.py
```

### Correct

```bash
python manage.py s3check
```

---

## Problem 2 — `manage.py: Permission denied`

If running as `ec2-user` produces:

```text
Permission denied
```

the project may belong to the `djangov2` user.

Switch to the application user:

```bash
sudo su - djangov2
```

Then:

```bash
cd /home/djangov2/techcake-django-app-v2
source venv/bin/activate
python manage.py s3check
```

---

# 🛰️ 16. IMDSv2 Troubleshooting

If this returns nothing:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

do not immediately assume the IAM Role is missing.

The EC2 instance may require **IMDSv2**.

Use:

```bash
TOKEN=$(curl -X PUT \
"http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

Then:

```bash
curl \
-H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Expected:

```text
shyamdev-dev-ec2-role
```

---

# 🛠️ 17. SSM Troubleshooting

The EC2 instance also uses:

```text
AmazonSSMManagedInstanceCore
```

for Systems Manager access.

During configuration, SSM temporarily showed an authorization error because the IAM permissions were being updated.

After restarting the SSM Agent, the logs confirmed:

```text
EC2RoleProvider Successfully connected with instance profile role credentials
```

The SSM instance subsequently returned to **Online**.

### Restart SSM Agent

```bash
sudo systemctl restart amazon-ssm-agent
```

### Check status

```bash
sudo systemctl status amazon-ssm-agent
```

Expected:

```text
Active: active (running)
```

---

# 🌐 18. SSM Network Connectivity Test

Test the SSM endpoint:

```bash
curl -I https://ssm.us-east-1.amazonaws.com
```

An HTTP response such as:

```text
400 Bad Request
```

can still confirm that the endpoint is reachable. The important distinction is between receiving an HTTP response and having a network timeout/connection failure.

---

# 🔍 19. Useful Verification Commands

### Check AWS identity

```bash
aws sts get-caller-identity
```

### Check IAM role through IMDSv2

```bash
TOKEN=$(curl -X PUT \
"http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl \
-H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

### Check SSM Agent

```bash
sudo systemctl status amazon-ssm-agent
```

### Check SSM logs

```bash
sudo journalctl -u amazon-ssm-agent -n 100 --no-pager
```

### Check S3 management command

```bash
python manage.py s3check
```

---

# 📸 20. Submission Screenshot Checklist

The following screenshots should be included with the task submission.

### Screenshot 1 — S3 Bucket

Show:

- Bucket name
- Region
- Block Public Access enabled

---

### Screenshot 2 — IAM Policy JSON

Show:

```text
shyamdev-dev-s3-policy
```

with:

```text
s3:ListBucket
s3:GetObject
s3:PutObject
```

---

### Screenshot 3 — IAM Role

Show:

```text
shyamdev-dev-ec2-role
```

and the attached policies:

```text
AmazonSSMManagedInstanceCore
shyamdev-dev-s3-policy
```

---

### Screenshot 4 — EC2 Instance

Show:

```text
IAM Role:
shyamdev-dev-ec2-role
```

This proves that the role is attached to the EC2 instance.

---

### Screenshot 5 — Application Configuration

Show the relevant S3 configuration:

```env
AWS_STORAGE_BUCKET_NAME=shyamdev-dev-s3
AWS_S3_REGION_NAME=us-east-1
```

Make sure no AWS Access Keys are visible.

---

### Screenshot 6 — Successful S3 Check

Show:

```bash
python manage.py s3check
```

with:

```text
S3 CHECK PASSED
```

and:

```text
Credentials were provided by the instance role (no access keys used).
```

---

# 🔐 Security Checklist

Before committing this project to GitHub:

- [ ] ❌ No AWS Access Key ID in source code
- [ ] ❌ No AWS Secret Access Key in source code
- [ ] ❌ No AWS Session Token in source code
- [ ] ❌ No passwords in `settings.py`
- [ ] ❌ No `.env` file committed
- [ ] ❌ No database credentials committed
- [ ] ❌ No private keys committed
- [ ] ✅ Use `.env.example` instead
- [ ] ✅ Use IAM Roles for AWS authentication

---

# 📄 Recommended `.gitignore`

Make sure the repository contains a `.gitignore` similar to:

```gitignore
# Python
__pycache__/
*.py[cod]
*.pyo

# Virtual environment
venv/
.venv/
env/

# Environment variables
.env
.env.*

# Keep example configuration
!.env.example

# Django
db.sqlite3
staticfiles/
media/

# Logs
*.log

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

> **Important:** Never commit `/home/djangov2/.env` or any file containing real credentials.

---

# 🧪 Final Verification Checklist

Before considering the task complete:

```text
☑ S3 bucket created
☑ Correct AWS region
☑ Block Public Access enabled
☑ Bucket Owner Enforced
☑ SSE-S3 encryption enabled
☑ Least-privilege S3 policy created
☑ s3:ListBucket configured
☑ s3:GetObject configured
☑ s3:PutObject configured
☑ S3 policy attached to EC2 IAM role
☑ SSM policy retained
☑ IAM role attached to EC2
☑ IMDSv2 working
☑ IAM Role credentials working
☑ AWS CLI identifies assumed role
☑ No AWS Access Keys in application
☑ Django S3Boto3Storage configured
☑ s3check command exists
☑ S3 write successful
☑ S3 read successful
☑ s3check passed
```

---

# 🏁 Final Result

The TechCake Django App v2 was successfully configured to access Amazon S3 securely through an **EC2 IAM Instance Role**.

The final architecture avoids permanent AWS credentials and follows the **Principle of Least Privilege**.

### Final Authentication

```text
EC2
 │
 │ Instance Profile
 ▼
shyamdev-dev-ec2-role
 │
 │ Temporary Credentials
 ▼
shyamdev-dev-s3
```

### Final Verification

```text
S3 CHECK PASSED
```

The application successfully performed both:

```text
WRITE → S3
READ  ← S3
```

using credentials supplied by the EC2 Instance Role.

---

## 📝 Key Lessons / Future Reference

### 1. EC2 can have only one attached Instance Role

If an EC2 instance already has an IAM Role for SSM, don't create another role just for S3.

Attach the required S3 policy to the existing role.

---

### 2. Never put AWS Access Keys in Django

Avoid:

```python
AWS_ACCESS_KEY_ID = "..."
AWS_SECRET_ACCESS_KEY = "..."
```

Use the IAM Role instead.

---

### 3. IMDSv2 requires a token

This may fail:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Use IMDSv2:

```bash
TOKEN=$(curl -X PUT \
"http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl \
-H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

---

### 4. `s3check.py` is a Django command

Don't execute:

```bash
./congrats/management/commands/s3check.py
```

Use:

```bash
python manage.py s3check
```

---

### 5. Successful STS identity verification

The expected identity should be an assumed role:

```text
arn:aws:sts::<ACCOUNT_ID>:assumed-role/shyamdev-dev-ec2-role/<INSTANCE_ID>
```

and **not**:

```text
arn:aws:iam::<ACCOUNT_ID>:user/<USERNAME>
```

---

### 6. Successful S3 verification

The strongest final proof is:

```text
S3 CHECK PASSED
```

together with:

```text
Credentials were provided by the instance role (no access keys used).
```

---

# 📚 Technologies Used

- **Amazon S3**
- **AWS IAM**
- **Amazon EC2**
- **AWS Systems Manager (SSM)**
- **Django**
- **django-storages**
- **boto3**
- **Python**
- **IMDSv2**

---

> **Task Status: ✅ COMPLETED**
>
> **S3 Integration: ✅**
>
> **IAM Role Authentication: ✅**
>
> **Least-Privilege Access: ✅**
>
> **No Static AWS Credentials: ✅**
>
> **S3 Write + Read Verification: ✅**
