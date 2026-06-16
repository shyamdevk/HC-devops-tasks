# 🌐Task 5 - WordPress Hosting with HTTPS on AlmaLinux 9

## 📖 Overview

This project demonstrates the deployment of a WordPress website on an AlmaLinux 9 EC2 instance using:

* Apache HTTP Server
* MariaDB Database Server
* WordPress CMS
* Let's Encrypt SSL Certificate
* Dedicated Linux User
* Dedicated Database User

The website is hosted under its own Linux user account and secured using HTTPS.

---

## 🏗️ Architecture

```text
Internet
    │
    ▼
Apache Virtual Host
    │
    ▼
WordPress
    │
    ▼
MariaDB Database
```

---

## ⚙️ Environment Details

| Component        | Version / Service  |
| ---------------- | ------------------ |
| Operating System | AlmaLinux 9        |
| Web Server       | Apache HTTP Server |
| Database         | MariaDB 10.5       |
| CMS              | WordPress          |
| SSL Provider     | Let's Encrypt      |
| Cloud Platform   | AWS EC2            |

---

## 👤 Dedicated User Creation

Created a dedicated Linux user for WordPress:

```bash
sudo useradd -m wordpress1
sudo passwd wordpress1
```

Home directory:

```text
/home/wordpress1
```

---

## 📦 WordPress Installation

Downloaded and extracted the latest WordPress package:

```bash
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
```

Installed under:

```text
/home/wordpress1/public_html
```

---

## 🗄️ Database Configuration

Created a dedicated database and user:

```sql
CREATE DATABASE wordpress1;

CREATE USER 'wordpress1_user'@'localhost'
IDENTIFIED BY 'password';

GRANT ALL PRIVILEGES
ON wordpress1.*
TO 'wordpress1_user'@'localhost';

FLUSH PRIVILEGES;
```

Updated WordPress configuration:

```php
define('DB_NAME', 'wordpress1');
define('DB_USER', 'wordpress1_user');
define('DB_PASSWORD', 'password');
define('DB_HOST', 'localhost');
```

---

## 🌍 Apache Virtual Host Configuration

Created virtual host:

```apache
<VirtualHost *:80>
    ServerName wordpress1.shyamdev.nixlabs.in

    DocumentRoot /home/wordpress1/public_html

    <Directory /home/wordpress1/public_html>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Restarted Apache:

```bash
sudo systemctl restart httpd
```

---

## 🔐 HTTPS Configuration

Installed Certbot:

```bash
sudo dnf install certbot python3-certbot-apache -y
```

Generated SSL certificate:

```bash
sudo certbot --apache -d wordpress1.shyamdev.nixlabs.in
```

Enabled automatic HTTP → HTTPS redirection.

---

## ✅ Verification

### Website

```text
https://wordpress1.shyamdev.nixlabs.in
```

### Admin Panel

```text
https://wordpress1.shyamdev.nixlabs.in/wp-admin
```

### SSL Renewal Test

```bash
sudo certbot renew --dry-run
```

---

## 🐞 Issues Encountered

### Issue 1: Homepage Displayed Instead of Login Page

**Problem**

Opening the website displayed the WordPress homepage instead of the login page.

**Cause**

WordPress loads the public homepage by default.

**Resolution**

Used:

```text
/wp-admin
```

or

```text
/ wp-login.php
```

to access the administrator login page.

---

### Issue 2: Website Timeout Error

**Problem**

Browser displayed:

```text
ERR_CONNECTION_TIMED_OUT
```

**Cause**

Network or service configuration issue.

**Resolution**

Verified:

* Apache service status
* DNS records
* Firewall rules
* AWS Security Group rules
* Virtual Host configuration

---

### Issue 3: Error Establishing a Database Connection

**Problem**

WordPress displayed:

```text
Error establishing a database connection
```

**Cause**

MariaDB service was not running.

**Resolution**

Checked service status:

```bash
sudo systemctl status mariadb
```

Reviewed MariaDB logs:

```bash
sudo tail -50 /var/log/mariadb/mariadb.log
```

---

### Issue 4: MariaDB Failed to Start

**Problem**

MariaDB service repeatedly failed.

**Error**

```text
Could not set the file size of './ibtmp1'
Probably out of disk space
```

**Root Cause**

Root filesystem was completely full.

```text
/dev/nvme0n1p4 100% used
```

**Resolution**

* Cleaned DNF cache
* Removed unnecessary files
* Freed disk space
* Planned EBS volume expansion for long-term stability

---

## 📊 Disk Usage Investigation

Checked disk usage:

```bash
df -h
```

Checked large files:

```bash
sudo find / -xdev -type f -size +50M
```

Identified major space consumers:

```text
/swapfile          2.0G
ib_logfile0        96M
```

---

## 📸 Screenshots

Add screenshots here before submission:

### Homepage


![Kubernetes](https://github.com/shyamdevk/HC-devops-tasks/blob/images/log.jpg)


### WordPress Dashboard


![Kubernetes](https://github.com/shyamdevk/HC-devops-tasks/blob/images/dash.jpg)


### HTTPS Certificate

```text
screenshots/https.png
```

---

## 🎯 Learning Outcomes

* Configured Apache Virtual Hosts
* Installed and configured WordPress
* Managed MariaDB databases and users
* Configured SSL using Let's Encrypt
* Troubleshot database connection issues
* Diagnosed disk space problems on Linux
* Managed DNS and AWS networking

---

## 👨‍💻 Author

**Shyam**

WordPress Hosting and HTTPS Configuration Lab on AlmaLinux 9
