# Task 3 - Static Website Hosting on AlmaLinux 9 with Apache and HTTPS

## Overview

This project demonstrates hosting a static website on an AlmaLinux 9 EC2 instance using:

* Apache HTTP Server
* Route 53 DNS
* Name-based Virtual Hosts
* Let's Encrypt SSL (HTTPS)
* Dedicated Linux user and document root

The website is served from a user-owned directory instead of the default Apache web root.

---

# Architecture

```text
User Browser
      |
      v
https://static1.shyamdev.nixlabs.in
      |
      v
Route 53 DNS
      |
      v
EC2 Instance (AlmaLinux 9)
      |
      v
Apache Virtual Host
      |
      v
/home/static1/public_html
```

---

# Prerequisites

* AWS Account
* EC2 Instance (AlmaLinux 9)
* Route 53 Hosted Zone
* SSH Access
* Static Website ZIP File
* Security Group allowing:

  * SSH (22)
  * HTTP (80)
  * HTTPS (443)

---

# Step 1: Connect to EC2

```bash
ssh -i key.pem ec2-user@<PUBLIC_IP>
```

Verify connection:

```bash
whoami
hostname
```

---

# Step 2: Create Dedicated User

Create a separate Linux user for website hosting.

```bash
sudo useradd -m static1
```

Verify:

```bash
id static1
```

Expected:

```text
uid=1001(static1) gid=1001(static1)
```

---

# Step 3: Create Website Directory

```bash
sudo mkdir -p /home/static1/public_html
```

Assign ownership:

```bash
sudo chown -R static1:static1 /home/static1/public_html
```

---

# Step 4: Transfer Website Files

From local machine:

```bash
scp -i key.pem ~/Downloads/static-site.zip ec2-user@<PUBLIC_IP>:/tmp/
```

Verify:

```bash
ls /tmp
```

---

# Step 5: Extract Website Files

Install unzip:

```bash
sudo dnf install unzip -y
```

Extract files:

```bash
sudo unzip /tmp/static-site.zip -d /home/static1/public_html/
```

Verify:

```bash
ls -la /home/static1/public_html
```

---

# Step 6: Install Apache

Install Apache HTTP Server:

```bash
sudo dnf install httpd -y
```

Enable and start service:

```bash
sudo systemctl enable --now httpd
```

Verify:

```bash
sudo systemctl status httpd
```

---

# Step 7: Configure Virtual Host

Create configuration file:

```bash
sudo vi /etc/httpd/conf.d/static1.conf
```

Add:

```apache
<VirtualHost *:80>
    ServerName static1.shyamdev.nixlabs.in

    DocumentRoot /home/static1/public_html

    <Directory /home/static1/public_html>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog logs/static1_error.log
    CustomLog logs/static1_access.log combined
</VirtualHost>
```

Verify configuration:

```bash
sudo httpd -t
```

Expected:

```text
Syntax OK
```

---

# Step 8: Configure Permissions

Grant Apache access:

```bash
sudo chmod 755 /home/static1
sudo chmod 755 /home/static1/public_html
```

Update ownership:

```bash
sudo chown -R static1:static1 /home/static1/public_html
```

---

# Step 9: Configure SELinux

Check status:

```bash
getenforce
```

Enable Apache access to user home directories:

```bash
sudo setsebool -P httpd_enable_homedirs 1
```

Apply context:

```bash
sudo chcon -R -t httpd_sys_content_t /home/static1/public_html
```

Restart Apache:

```bash
sudo systemctl restart httpd
```

---

# Step 10: Configure Route 53 DNS

Create A Record:

| Record Name | Type | Value         |
| ----------- | ---- | ------------- |
| static1     | A    | EC2 Public IP |

Verify DNS:

```bash
nslookup static1.shyamdev.nixlabs.in
```

Expected:

```text
Name: static1.shyamdev.nixlabs.in
Address: <PUBLIC_IP>
```

---

# Step 11: Open Required Ports

Security Group Inbound Rules:

| Type  | Port |
| ----- | ---- |
| SSH   | 22   |
| HTTP  | 80   |
| HTTPS | 443  |

---

# Step 12: Install SSL Certificate

Install Certbot:

```bash
sudo dnf install epel-release -y
sudo dnf install certbot python3-certbot-apache -y
```

Generate certificate:

```bash
sudo certbot --apache -d static1.shyamdev.nixlabs.in
```

Provide:

* Email Address
* Accept Terms of Service
* Enable HTTP to HTTPS Redirect

---

# Step 13: Verify HTTPS

Open:

```text
https://static1.shyamdev.nixlabs.in
```

Checks:

* Website loads successfully
* SSL certificate is valid
* Browser shows secure connection
* HTTP redirects to HTTPS

---

# Step 14: Verify Certificate Renewal

Test renewal:

```bash
sudo certbot renew --dry-run
```

Expected:

```text
Congratulations, all simulated renewals succeeded
```

---

# Troubleshooting

## Issue 1: DNS Not Resolving

### Error

```text
NXDOMAIN
```

### Cause

Incorrect Route 53 record or DNS propagation delay.

### Fix

Verified A record and waited for DNS propagation.

---

## Issue 2: Apache Test Page Displayed

### Error

Default AlmaLinux page appeared.

### Cause

Apache default welcome page was being served.

### Fix

Disabled default configuration and verified Virtual Host settings.

---

## Issue 3: 403 Forbidden

### Error

```text
403 Forbidden
```

### Cause

Apache lacked permission to access files in user home directory.

### Fix

Updated:

* Directory permissions
* Ownership
* SELinux context

---

## Issue 4: SSL Configuration Error

### Error

```text
SSLCertificateFile does not exist
```

### Cause

Invalid SSL configuration file.

### Fix

Removed incorrect SSL configuration and reran Certbot.

---

# Verification Commands

```bash
sudo httpd -t
```

```bash
sudo apachectl -S
```

```bash
sudo systemctl status httpd
```

```bash
curl http://127.0.0.1
```

```bash
sudo certbot renew --dry-run
```

---

# Final Result

| Component               | Status    |
| ----------------------- | --------- |
| Apache Installed        | Completed |
| Virtual Host Configured | Completed |
| Website Deployed        | Completed |
| Route 53 DNS Configured | Completed |
| HTTPS Enabled           | Completed |
| Auto Renewal Verified   | Completed |

![Kubernetes](https://github.com/shyamdevk/HC-devops-tasks/blob/images/live.png)

---

# Live URL

```text
https://static1.shyamdev.nixlabs.in
```

---


