# Task 2 — Promote the Data Tier to Managed Services

## Amazon RDS + Amazon ElastiCache

> Migration of a Django application's database and cache from locally hosted services on Amazon EC2 to AWS managed services, followed by moving the managed data tier into private subnets.

---

## Table of Contents

- [Overview](#overview)
- [Task Requirements](#task-requirements)
- [Initial Environment](#initial-environment)
- [Initial Architecture](#initial-architecture)
- [Phase 1 — Amazon RDS Migration](#phase-1--amazon-rds-migration)
- [Phase 2 — Amazon ElastiCache Migration](#phase-2--amazon-elasticache-migration)
- [Phase 3 — Local Service Removal](#phase-3--local-service-removal)
- [Phase 4 — Private Subnet Architecture](#phase-4--private-subnet-architecture)
- [Phase 5 — RDS Migration to Private Subnets](#phase-5--rds-migration-to-private-subnets)
- [Phase 6 — ElastiCache Migration to Private Subnets](#phase-6--elasticache-migration-to-private-subnets)
- [Django Configuration](#django-configuration)
- [Database Verification](#database-verification)
- [Redis Verification](#redis-verification)
- [Security Configuration](#security-configuration)
- [EC2 Cleanup](#ec2-cleanup)
- [Final Architecture](#final-architecture)
- [Final Resource Configuration](#final-resource-configuration)
- [Submission Evidence](#submission-evidence)
- [Important Commands](#important-commands)
- [Troubleshooting](#troubleshooting)
- [Final Checklist](#final-checklist)
- [Lessons Learned](#lessons-learned)

---

# Overview

The purpose of this task was to promote the application's data tier from locally managed services to AWS managed services.

The original application used:

- Local MariaDB on the EC2 instance
- Local Redis on the EC2 instance

The application was migrated to:

- Amazon RDS for the database
- Amazon ElastiCache for Redis

After the initial migration, the network architecture was improved by creating private subnets inside the existing default VPC and deploying the managed data services into those private subnets.

The final architecture keeps the EC2 application server accessible through the public layer while the database and cache remain private.

---

# Task Requirements

The task required the following:

## Database — Amazon RDS

- Launch an Amazon RDS MySQL/MariaDB-compatible database.
- Keep public access disabled.
- Place RDS in the application VPC.
- Configure a Security Group whose inbound rule references the EC2 instance Security Group rather than a public CIDR.
- Create a dedicated remote database user and password for the application.
- Use least privilege for the application database user.
- Do not use the RDS master user from Django.
- Create a recognizable test record before migration.
- Migrate the local MariaDB data into RDS.
- Update the Django application to use the RDS endpoint.
- Remove the local MariaDB service.

## Cache — Amazon ElastiCache

- Create a small Redis/Valkey cluster.
- Use cluster mode disabled.
- Use a single-node configuration.
- Do not use public access.
- Place the cache inside the application VPC.
- Allow Redis traffic only from the EC2 Security Group.
- Configure Django to use:

```text
redis://<primary-endpoint>:6379/0
```

- Remove the local Redis service.
- Verify the application continues to work using ElastiCache.

---

# Initial Environment

## Application Server

| Component | Configuration |
|---|---|
| Operating System | AlmaLinux 9 |
| Application | Django |
| Web Server | Nginx |
| Application Server | Gunicorn |
| Gunicorn Service | `/etc/systemd/system/gunicorn.service` |
| Project Directory | `/home/djangov2/techcake-django-app-v2` |
| Virtual Environment | `/home/djangov2/techcake-django-app-v2/venv` |
| Environment File | `/home/djangov2/.env` |
| HTTPS | Enabled |
| SELinux | Enforcing |
| Remote Management | AWS Systems Manager Session Manager |

## Application URL

```text
https://djangov2.shyamdev.nixlabs.in
```

---

# Initial Data Tier

Before migration:

```text
Django
   |
   +---- Local MariaDB
   |
   +---- Local Redis
```

The EC2 instance was originally responsible for running the application as well as the database and cache.

---

# Initial Architecture

```text
                         Internet
                             |
                           HTTPS
                             |
                         Nginx
                             |
                         Gunicorn
                             |
                          Django
                       /           \
                      /             \
                     v               v
             Local MariaDB      Local Redis
                EC2               EC2
```

The objective was to remove the database and cache management responsibility from the EC2 instance.

---

# Phase 1 — Amazon RDS Migration

## 1. Create Amazon RDS

A MySQL-compatible Amazon RDS instance was created.

Initial configuration included:

```text
Engine:
MySQL

Instance:
db.t3.micro

Storage:
20 GB

Deployment:
Single-AZ

Port:
3306
```

---

# 2. RDS Security Group

A dedicated RDS Security Group was configured.

Required inbound rule:

```text
Type:       MySQL/Aurora
Protocol:   TCP
Port:       3306
Source:     EC2 Security Group
```

The database was not opened to:

```text
0.0.0.0/0
```

The purpose of using a Security Group reference is to allow only the application server to reach the database.

---

# 3. Create Dedicated Application Database User

The Django application uses a dedicated database user:

```text
djangoapp
```

Database:

```text
djangodb
```

The application does not use the RDS master account.

The intended permission scope is the application database only:

```sql
GRANT ALL PRIVILEGES
ON djangodb.*
TO 'djangoapp'@'%';
```

The important principle is that the application user should not have unrestricted privileges on every database.

The master account remains an administrative account and is not used by Django.

---

# 4. Database Migration

The local MariaDB database was exported using a SQL dump.

The dump was imported into Amazon RDS.

The migration transferred the existing Django database structure and application data.

A recognizable Django user was also available as a migration verification record.

The primary Django administrator is:

```text
shyamdev
```

The account is configured as:

```text
Superuser: True
Staff:     True
```

This user is sufficient as the recognizable migrated record required by the task.

A temporary migration verification user named:

```text
migrationtest
```

was also created during testing and could be removed after the migration was verified.

---

# 5. Django RDS Configuration

The Django environment file is:

```text
/home/djangov2/.env
```

Database configuration follows this structure:

```env
DB_NAME=djangodb
DB_USER=djangoapp
DB_PASSWORD=<REDACTED>
DB_HOST=<RDS-ENDPOINT>
DB_PORT=3306
```

The real password must never be committed to GitHub.

---

# 6. Verify Django Uses RDS

Activate the virtual environment:

```bash
cd /home/djangov2/techcake-django-app-v2
source venv/bin/activate
```

Open the Django shell:

```bash
python manage.py shell
```

Run:

```python
from django.db import connection

print(connection.settings_dict["HOST"])
print(connection.settings_dict["NAME"])
print(connection.settings_dict["USER"])
```

Expected values:

```text
HOST = <Amazon RDS endpoint>
NAME = djangodb
USER = djangoapp
```

This verifies that Django is not using a local database.

---

# 7. Test SQL Connectivity

From the Django shell:

```python
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT NOW();")
    print(cursor.fetchone())
```

A timestamp returned from the query confirms successful communication between Django and RDS.

---

# 8. Verify Django Migrations

Run:

```bash
python manage.py showmigrations
```

All required migrations should show:

```text
[X]
```

Then:

```bash
python manage.py migrate
```

Successful verification:

```text
No migrations to apply.
```

This confirms that the RDS database contains the expected Django schema.

---

# Phase 2 — Amazon ElastiCache Migration

## 1. Create Redis Cluster

Amazon ElastiCache was created using Redis.

The intended configuration is:

```text
Deployment:
Node-based cluster

Cluster Mode:
Disabled

Primary Nodes:
1

Replicas:
0

Port:
6379
```

The single-node configuration matches the task requirement.

---

# 2. ElastiCache Security Group

A dedicated Redis Security Group was configured.

Required inbound rule:

```text
Protocol: TCP
Port:     6379
Source:   EC2 Security Group
```

Redis should not be exposed using:

```text
0.0.0.0/0
```

---

# 3. Redis Encryption Configuration

The final configuration uses:

```text
Encryption at Rest:
Enabled
```

and:

```text
Encryption in Transit:
Disabled
```

Because encryption in transit is disabled, the Django connection uses:

```text
redis://
```

rather than:

```text
rediss://
```

If encryption in transit were enabled, the application would require `rediss://` and additional TLS configuration.

---

# 4. Django Redis Configuration

The Django cache URL uses the ElastiCache primary endpoint:

```env
REDIS_URL=redis://<ELASTICACHE-PRIMARY-ENDPOINT>:6379/0
```

The `/0` specifies Redis database 0.

---

# 5. Verify Redis Configuration

Open the Django shell:

```bash
python manage.py shell
```

Run:

```python
from django.conf import settings

print(settings.CACHES)
```

Expected configuration contains:

```text
django.core.cache.backends.redis.RedisCache
```

and:

```text
redis://<ElastiCache-endpoint>:6379/0
```

The important point is that the endpoint belongs to Amazon ElastiCache rather than:

```text
127.0.0.1
```

or:

```text
localhost
```

---

# 6. Verify Redis Read/Write

Run:

```python
from django.core.cache import cache

cache.set("redis_test", "hello", 60)
print(cache.get("redis_test"))
```

Expected:

```text
hello
```

This verifies that Django can:

1. Connect to Redis.
2. Write a cache value.
3. Read the value back.

---

# Phase 3 — Local Service Removal

After RDS and ElastiCache were successfully verified, the local data services were removed.

---

# MariaDB Removal

The local MariaDB service was removed.

Verification:

```bash
systemctl status mariadb
```

Expected:

```text
Unit mariadb.service could not be found.
```

The local MySQL/MariaDB port was also checked:

```bash
sudo ss -tulpn | grep 3306
```

No output confirms that nothing is listening on local TCP port 3306.

---

# Local MySQL Socket Verification

Running:

```bash
sudo mysql -u root -p
```

or:

```bash
sudo mysql -u shyamdev -p
```

returns:

```text
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2)
```

This is expected after removing the local database server.

It means the MySQL client attempted to connect to a local MySQL/MariaDB socket, but no local database server exists.

It does not indicate that Amazon RDS is unavailable.

---

# Redis Removal

The local Redis service was also removed.

The application configuration was changed so that Django communicates directly with Amazon ElastiCache.

The local Redis service is no longer required.

---

# Phase 4 — Private Subnet Architecture

After the initial managed-service migration, the architecture was improved.

The default VPC originally contained public subnets.

Private subnets were created inside the same default VPC.

The objective was:

```text
EC2
Public Subnet

RDS
Private Subnet

ElastiCache
Private Subnet
```

All three resources remain inside the same VPC.

---

# VPC

The default VPC is:

```text
Default VPC
```

The VPC contains eight subnets after the private subnets were created.

The newly created private subnets are:

```text
hc-dev-subnet-private1-us-east-1a
hc-dev-subnet-private2-us-east-1b
```

They are located in:

```text
us-east-1a
us-east-1b
```

---

# Private Subnet CIDRs

The private subnet CIDRs used for the managed services are:

```text
172.31.100.0/24
172.31.101.0/24
```

These subnets were used for the RDS and ElastiCache subnet groups.

---

# Private Route Table

A dedicated private route table was created:

```text
hc-dev-private-rt
```

The private subnets are associated with this route table.

The private route table should not route directly to the Internet Gateway.

Expected basic route:

```text
Destination: VPC CIDR
Target:      local
```

A NAT Gateway can be used if private resources require outbound Internet connectivity, but RDS and ElastiCache do not require direct Internet access for normal operation.

---

# Private Subnet Public IP Configuration

The private subnets should have:

```text
Auto-assign public IPv4:
Disabled
```

This prevents resources launched into these subnets from automatically receiving public IPv4 addresses.

---

# Phase 5 — RDS Migration to Private Subnets

An existing RDS instance cannot simply be treated as though its network location has changed without considering the subnet group and deployment behavior.

To safely migrate the database into private subnets, the existing database was protected using an RDS snapshot.

---

# 1. Create RDS Snapshot

A manual snapshot was created before creating the new private RDS instance.

Snapshot:

```text
shyamdev-rds-before-private
```

Purpose:

- Preserve the database.
- Provide rollback capability.
- Create a new RDS instance from the existing data.
- Avoid losing the only AWS copy of the database after local MariaDB was removed.

---

# 2. Create RDS DB Subnet Group

A new DB subnet group was created:

```text
shyamdev-private-db-subnet-group
```

It contains two private subnets in different Availability Zones.

```text
Private Subnet 1
us-east-1a

Private Subnet 2
us-east-1b
```

The subnet group is associated with the default VPC.

---

# 3. Restore RDS from Snapshot

The RDS snapshot was restored into a new DB instance.

New DB instance identifier:

```text
shyamdev-dev-rds02
```

Important configuration:

```text
Engine:
MySQL Community

Deployment:
Single-AZ

Instance:
db.t3.micro

Storage:
20 GB

VPC:
Default VPC

DB Subnet Group:
shyamdev-private-db-subnet-group

Public Access:
No

Security Group:
shyamdev-dev-rds01-sg
```

The new instance was restored from the existing snapshot, preserving the database contents.

---

# 4. Why a New RDS Instance Was Used

The original RDS instance was kept available during the migration.

A new instance was restored from the snapshot rather than immediately deleting the existing database.

This provided a safe migration path:

```text
Existing RDS
     |
     | Snapshot
     v
New Private RDS
     |
     | Verify
     v
Update Django
     |
     | Verify
     v
Delete old RDS only after confirmation
```

This approach minimizes the risk of losing the only remaining database copy.

---

# 5. RDS Public Access

The restored RDS instance was configured with:

```text
Public access:
No
```

This means the RDS instance does not receive a public IP address.

It can be reached by resources inside the VPC according to Security Group rules.

---

# 6. RDS Security Group

The RDS Security Group remains restricted to the EC2 Security Group.

Expected inbound rule:

```text
TCP 3306
Source: EC2 Security Group
```

There should be no rule such as:

```text
TCP 3306
0.0.0.0/0
```

---

# Phase 6 — ElastiCache Migration to Private Subnets

The Redis cache was also moved into private networking.

Because the existing ElastiCache deployment needed to be recreated with the desired subnet placement, a backup was created before the migration.

---

# 1. ElastiCache Backup

A backup was created:

```text
shyamdev-cac01-before-private
```

This backup was used as the recovery source for the new private Redis cluster.

---

# 2. Create Private Redis Subnet Group

A dedicated ElastiCache subnet group was created:

```text
shyamdev-private-redis-sg
```

Associated subnets:

```text
us-east-1a
CIDR: 172.31.100.0/24

us-east-1b
CIDR: 172.31.101.0/24
```

Both subnets belong to the same default VPC.

---

# 3. Restore/Create New Redis Cluster

The new Redis cluster uses:

```text
Engine:
Redis

Deployment:
Node-based cluster

Cluster Mode:
Disabled

Primary:
1

Replicas:
0

Port:
6379

Node Type:
cache.t3.micro

Subnet Group:
shyamdev-private-redis-sg
```

The single-node configuration matches the task requirement.

---

# 4. Multi-AZ

The task requires a single-node Redis configuration.

Therefore:

```text
Multi-AZ:
Disabled

Replicas:
0
```

A configuration such as:

```text
1 Primary
2 Replicas
```

would not match the single-node requirement and would unnecessarily increase resource usage and cost.

---

# 5. Redis Encryption

Final configuration:

```text
Encryption at Rest:
Enabled

Encryption in Transit:
Disabled
```

Therefore the application continues using:

```text
redis://<primary-endpoint>:6379/0
```

rather than:

```text
rediss://
```

---

# 6. Redis Security Group

The Redis Security Group:

```text
shyamdev-dev-cac01-sg
```

should allow:

```text
TCP 6379
Source: EC2 Security Group
```

No public CIDR should be allowed.

---

# Django Configuration

The application configuration is stored in:

```text
/home/djangov2/.env
```

The database configuration follows:

```env
DB_NAME=djangodb
DB_USER=djangoapp
DB_PASSWORD=<REDACTED>
DB_HOST=<PRIVATE-RDS-ENDPOINT>
DB_PORT=3306
```

Redis:

```env
REDIS_URL=redis://<PRIVATE-ELASTICACHE-ENDPOINT>:6379/0
```

The `.env` file must never be committed to GitHub.

---

# Gunicorn

After changing the environment configuration, Gunicorn must be restarted so that the application loads the updated values.

```bash
sudo systemctl restart gunicorn
```

Check:

```bash
sudo systemctl status gunicorn
```

Expected:

```text
active (running)
```

---

# Nginx

Nginx continues to provide the HTTPS frontend.

Check:

```bash
sudo systemctl status nginx
```

Expected:

```text
active (running)
```

---

# Database Verification

## Verify RDS Host

```bash
python manage.py shell
```

```python
from django.db import connection

print(connection.settings_dict["HOST"])
```

Expected:

```text
<Amazon RDS endpoint>
```

---

## Verify Database User

```python
print(connection.settings_dict["USER"])
```

Expected:

```text
djangoapp
```

---

## Verify Database Name

```python
print(connection.settings_dict["NAME"])
```

Expected:

```text
djangodb
```

---

## Verify SQL

```python
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT NOW();")
    print(cursor.fetchone())
```

A returned timestamp confirms successful database connectivity.

---

# Migration Verification

Run:

```bash
python manage.py showmigrations
```

All required migrations should show:

```text
[X]
```

Then:

```bash
python manage.py migrate
```

Expected:

```text
No migrations to apply.
```

A Django warning about MySQL Strict Mode may appear:

```text
mysql.W002
MySQL Strict Mode is not set
```

This is a Django warning rather than a migration failure.

The migration can still complete successfully.

---

# Redis Verification

## Verify Cache Backend

```bash
python manage.py shell
```

```python
from django.conf import settings

print(settings.CACHES)
```

Expected structure:

```text
{
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://<ElastiCache-endpoint>:6379/0'
    }
}
```

---

# Test Redis Read/Write

```python
from django.core.cache import cache

cache.set("redis_test", "hello", 60)
print(cache.get("redis_test"))
```

Expected:

```text
hello
```

This proves that Django can communicate with Amazon ElastiCache.

---

# Optional Direct Redis Test

If `redis-cli` is installed:

```bash
redis-cli -h <ELASTICACHE-ENDPOINT> -p 6379 ping
```

Expected:

```text
PONG
```

If `redis-cli` is not installed, it is not necessary to install it for this task.

The Django cache test already proves application-level connectivity.

---

# Application Verification

Check the website:

```text
https://djangov2.shyamdev.nixlabs.in
```

Verify:

- Homepage loads.
- HTTPS works.
- Django application works.
- Django Admin works.
- Database-backed pages work.
- Cache-dependent functionality works.

---

# Django Admin Verification

The main administrator account is:

```text
shyamdev
```

Current privileges:

```text
Superuser: True
Staff:     True
```

The task only requires a recognizable migrated record.

A second non-superuser account is not mandatory.

The temporary:

```text
migrationtest
```

account was used for migration verification and can be removed after evidence has been collected.

---

# EC2 Cleanup

After successful migration, temporary migration files were removed.

Examples include:

```text
django-backup.sql
database-backups/
```

These were local database migration artifacts and were no longer required after successful RDS migration and snapshot creation.

---

# Files That Must Be Kept

The following application files/directories must remain:

```text
/home/djangov2/.env

/home/djangov2/techcake-django-app-v2/

/home/djangov2/techcake-django-app-v2/venv/
```

The Python virtual environment must not be removed because it contains the application's installed dependencies.

---

# Python Cache Cleanup

Python cache files can safely be removed because Python recreates them when necessary.

```bash
sudo find /home/djangov2/techcake-django-app-v2 \
-type d \
-name "__pycache__" \
-exec rm -rf {} +
```

Compiled Python files can also be removed:

```bash
sudo find /home/djangov2/techcake-django-app-v2 \
-type f \
\( -name "*.pyc" -o -name "*.pyo" \) \
-delete
```

---

# Files Not to Delete

Do not delete package data files inside:

```text
venv/lib/python3.9/site-packages/
```

Files ending in:

```text
.gz
.tar.gz
.json.gz
```

may be part of installed Python packages.

Removing them manually can break the virtual environment or application.

---

# Final Architecture

```text
                              Internet
                                  |
                                  |
                                HTTPS
                                  |
                                  v
                         +----------------+
                         |      EC2       |
                         |                |
                         | Public Subnet  |
                         |                |
                         | Nginx          |
                         | Gunicorn       |
                         | Django         |
                         +-------+--------+
                                 |
                  Security Group References
                                 |
                +----------------+----------------+
                |                                 |
                v                                 v
       +-------------------+             +-------------------+
       |    Amazon RDS     |             |  Amazon           |
       |      MySQL        |             |  ElastiCache      |
       |                   |             |      Redis        |
       |  Private Subnet   |             |  Private Subnet   |
       |                   |             |                   |
       | TCP 3306          |             | TCP 6379          |
       +-------------------+             +-------------------+
```

---

# Network Architecture

```text
Default VPC
|
+-- Public Subnets
|      |
|      +-- EC2
|
+-- Private Subnet
|      |
|      +-- Amazon RDS
|
+-- Private Subnet
       |
       +-- Amazon ElastiCache
```

Private subnets:

```text
hc-dev-subnet-private1-us-east-1a
172.31.100.0/24

hc-dev-subnet-private2-us-east-1b
172.31.101.0/24
```

Private route table:

```text
hc-dev-private-rt
```

---

# Final Resource Configuration

## EC2

```text
Name:
shyamdev-dev-web01

OS:
AlmaLinux 9

Role:
Django Application Server

Network:
Public Subnet

Services:
Nginx
Gunicorn
Django
```

---

## RDS

```text
Identifier:
shyamdev-dev-rds02

Engine:
MySQL

Instance:
db.t3.micro

Storage:
20 GB

Deployment:
Single-AZ

Database:
djangodb

Application User:
djangoapp

Port:
3306

Public Access:
No

Subnet Group:
shyamdev-private-db-subnet-group

Network:
Private Subnets
```

---

## ElastiCache

```text
Engine:
Redis

Deployment:
Node-based Cluster

Cluster Mode:
Disabled

Primary Nodes:
1

Replicas:
0

Multi-AZ:
Disabled

Node Type:
cache.t3.micro

Port:
6379

Subnet Group:
shyamdev-private-redis-sg

Network:
Private Subnets

Encryption at Rest:
Enabled

Encryption in Transit:
Disabled
```

---

# Security Configuration

## EC2 Security Group

Public application access is restricted according to the existing infrastructure policy.

Expected public services:

```text
HTTP  80
HTTPS 443
```

SSH is restricted to trusted sources according to the environment policy.

---

# RDS Security Group

```text
TCP 3306
Source:
EC2 Security Group
```

No unrestricted public database access.

---

# ElastiCache Security Group

```text
TCP 6379
Source:
EC2 Security Group
```

No unrestricted public Redis access.

---

# Data Protection

The migration was performed using AWS-managed backups/snapshots to avoid losing the database after the local MariaDB instance had already been removed.

RDS snapshot:

```text
shyamdev-rds-before-private
```

ElastiCache backup:

```text
shyamdev-cac01-before-private
```

These backups provided rollback/recovery options while creating the new private managed-service resources.

---

# Submission Evidence

The following screenshots/evidence should be retained for the GitHub report or task submission.

## RDS

### RDS Instance

Show:

- DB identifier
- Engine
- Status
- Endpoint
- VPC
- Public Access = No
- DB Subnet Group

### RDS Subnet Group

Show:

```text
shyamdev-private-db-subnet-group
```

and both private subnets.

### RDS Security Group

Show:

```text
TCP 3306
Source: EC2 Security Group
```

### Dedicated DB User

Connect using the RDS master account and run:

```sql
SELECT User, Host FROM mysql.user;
```

Then:

```sql
SHOW GRANTS FOR 'djangoapp'@'%';
```

This demonstrates that:

- `djangoapp` exists.
- Django uses a dedicated application account.
- The account has database-specific permissions.

### Django RDS Verification

Show:

```python
print(connection.settings_dict["HOST"])
print(connection.settings_dict["USER"])
print(connection.settings_dict["NAME"])
```

### Migration Verification

Show:

```bash
python manage.py migrate
```

with:

```text
No migrations to apply.
```

---

# ElastiCache Evidence

### Cluster

Show:

- Redis engine
- Cluster mode disabled
- Single primary node
- No replicas
- Port 6379
- Private subnet group

### ElastiCache Subnet Group

Show:

```text
shyamdev-private-redis-sg
```

and both private subnets.

### ElastiCache Security Group

Show:

```text
TCP 6379
Source: EC2 Security Group
```

### Django Redis Configuration

Show:

```python
from django.conf import settings
print(settings.CACHES)
```

with the ElastiCache endpoint.

### Redis Test

Show:

```python
from django.core.cache import cache

cache.set("redis_test", "hello", 60)
print(cache.get("redis_test"))
```

Expected:

```text
hello
```

---

# Private Subnet Evidence

Show the VPC resource map containing:

```text
Default VPC
|
+-- Public Subnets
|
+-- hc-dev-subnet-private1-us-east-1a
|
+-- hc-dev-subnet-private2-us-east-1b
|
+-- hc-dev-private-rt
```

Also provide evidence that:

```text
Auto-assign Public IPv4 = Disabled
```

for the private subnets.

---

# Local Service Removal Evidence

## MariaDB

```bash
systemctl status mariadb
```

Expected:

```text
Unit mariadb.service could not be found.
```

Also:

```bash
sudo ss -tulpn | grep 3306
```

Expected:

```text
No output
```

---

# Important Commands Reference

## Activate Django Environment

```bash
cd /home/djangov2/techcake-django-app-v2
source venv/bin/activate
```

---

## Django Shell

```bash
python manage.py shell
```

---

## Check Database

```python
from django.db import connection

print(connection.settings_dict["HOST"])
print(connection.settings_dict["NAME"])
print(connection.settings_dict["USER"])
```

---

## Test Database

```python
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT NOW();")
    print(cursor.fetchone())
```

---

## Check Migrations

```bash
python manage.py showmigrations
```

```bash
python manage.py migrate
```

---

## Check Redis Configuration

```python
from django.conf import settings

print(settings.CACHES)
```

---

## Test Redis

```python
from django.core.cache import cache

cache.set("redis_test", "hello", 60)
print(cache.get("redis_test"))
```

---

## Check Gunicorn

```bash
sudo systemctl status gunicorn
```

---

## Restart Gunicorn

```bash
sudo systemctl restart gunicorn
```

---

## Check Nginx

```bash
sudo systemctl status nginx
```

---

## Check Local Database Port

```bash
sudo ss -tulpn | grep 3306
```

---

## Check Local Redis Port

```bash
sudo ss -tulpn | grep 6379
```

---

# Troubleshooting

## Django Cannot Connect to RDS

Check:

```bash
cat /home/djangov2/.env
```

Do not expose the password when taking screenshots.

Verify:

```text
DB_HOST
DB_NAME
DB_USER
DB_PORT
```

Then verify the RDS Security Group:

```text
TCP 3306
Source: EC2 Security Group
```

Also confirm EC2 and RDS are in the same VPC.

Restart Gunicorn:

```bash
sudo systemctl restart gunicorn
```

---

# Django Cannot Connect to Redis

Check:

```python
from django.conf import settings
print(settings.CACHES)
```

The endpoint should be the ElastiCache endpoint.

It should not be:

```text
localhost
```

or:

```text
127.0.0.1
```

Verify the Redis Security Group:

```text
TCP 6379
Source: EC2 Security Group
```

Restart Gunicorn:

```bash
sudo systemctl restart gunicorn
```

---

# Local MySQL Socket Error

If:

```bash
mysql -u root -p
```

returns:

```text
ERROR 2002 (HY000):
Can't connect to local MySQL server through socket
```

this is expected if local MariaDB has been removed.

The command attempts to connect to the local socket.

To connect to RDS, specify the remote host:

```bash
mysql -h <RDS-ENDPOINT> -u <USERNAME> -p
```

---

# Redis CLI Not Found

If:

```bash
redis-cli
```

returns:

```text
-bash: redis-cli: command not found
```

this only means the Redis client utility is not installed.

It does not mean ElastiCache is unavailable.

Django itself can be used to verify Redis:

```python
from django.core.cache import cache

cache.set("test", "success", 60)
print(cache.get("test"))
```

Expected:

```text
success
```

---

# Important Security Notes

Never commit:

```text
.env
```

or any file containing:

- Database passwords
- AWS credentials
- API keys
- Secret keys
- Private keys
- Tokens

Use an example configuration for GitHub:

```env
DB_NAME=djangodb
DB_USER=djangoapp
DB_PASSWORD=<REDACTED>
DB_HOST=<RDS-ENDPOINT>
DB_PORT=3306

REDIS_URL=redis://<ELASTICACHE-ENDPOINT>:6379/0
```

---

# Final Checklist

## Application

- [x] Django application working
- [x] Nginx working
- [x] Gunicorn working
- [x] HTTPS working
- [x] Django Admin working

## Amazon RDS

- [x] RDS created
- [x] Database migrated
- [x] Dedicated application user created
- [x] Application uses `djangoapp`
- [x] Database name verified
- [x] RDS connectivity verified
- [x] SQL query verified
- [x] Django migrations verified
- [x] Public access disabled
- [x] Private DB subnet group created
- [x] Private RDS instance created from snapshot
- [x] RDS Security Group restricted to EC2 Security Group

## Amazon ElastiCache

- [x] Redis created
- [x] Cluster mode disabled
- [x] Single-node configuration
- [x] Replicas = 0
- [x] Multi-AZ disabled
- [x] Port 6379
- [x] Private subnet group created
- [x] Redis Security Group restricted to EC2 Security Group
- [x] Encryption at rest enabled
- [x] Encryption in transit disabled
- [x] Django Redis URL verified
- [x] Cache write/read verified

## Networking

- [x] Default VPC used
- [x] Private subnets created
- [x] Private route table created
- [x] Private subnets associated with private route table
- [x] Public IP assignment disabled for private subnets
- [x] RDS placed in private subnet group
- [x] ElastiCache placed in private subnet group

## Cleanup

- [x] Local MariaDB removed
- [x] Local Redis removed
- [x] Local database dump removed after successful migration
- [x] Temporary migration files removed
- [x] Project retained
- [x] Python virtual environment retained
- [x] Application environment configuration retained securely

---

# Final Result

The Django application has been successfully migrated from locally managed data services to AWS managed services.

The final data flow is:

```text
Django Application
       |
       +--------------------+
       |                    |
       v                    v
 Amazon RDS          Amazon ElastiCache
   MySQL                  Redis
       |                    |
       +---- Private -------+
            Subnets
```

The EC2 instance remains responsible for:

```text
Nginx
Gunicorn
Django
```

while AWS manages:

```text
Database:
Amazon RDS

Cache:
Amazon ElastiCache
```

The final design removes the operational dependency on local MariaDB and Redis and keeps the managed data tier private within the VPC.

---

# Key Takeaways

## 1. RDS Replaces Local MariaDB

The application no longer depends on a database process running on the EC2 instance.

Instead:

```text
Django → Amazon RDS
```

---

## 2. ElastiCache Replaces Local Redis

The application no longer depends on Redis running locally.

Instead:

```text
Django → Amazon ElastiCache
```

---

## 3. Dedicated Application Database User

Django uses:

```text
djangoapp
```

instead of the RDS master account.

This follows the principle of least privilege.

---

## 4. Private Data Tier

The application server can remain publicly reachable while the database and cache remain private.

```text
Public:
EC2

Private:
RDS
ElastiCache
```

---

## 5. Security Group Referencing

Instead of allowing:

```text
0.0.0.0/0
```

the managed services accept traffic from the EC2 Security Group.

This provides controlled service-to-service communication.

---

## 6. Snapshots Provide a Safe Migration Path

The RDS snapshot and ElastiCache backup provided recovery options before moving the services into private subnets.

The migration approach was:

```text
Existing Managed Service
          |
          v
       Snapshot
          |
          v
New Private Managed Service
          |
          v
Update Application
          |
          v
Verify
          |
          v
Remove Old Resource
```

---

# Final Status

```text
Task 2
Status: Completed

Application:
Django + Nginx + Gunicorn

Database:
Amazon RDS MySQL

Cache:
Amazon ElastiCache Redis

Network:
Default VPC

Application:
Public Subnet

Database:
Private Subnet

Cache:
Private Subnet

Local MariaDB:
Removed

Local Redis:
Removed

RDS Public Access:
Disabled

ElastiCache Public Access:
Disabled
```

The application successfully operates using Amazon RDS and Amazon ElastiCache while the managed data services remain isolated inside private subnets.
````
