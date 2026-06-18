# 🚀 Task 7a – WordPress Migration (wordpress1 → wordpress2)

## 📋 Objective

Migrate an existing WordPress website from **wordpress1** to **wordpress2** by:

* Creating a dedicated Linux user
* Migrating website files
* Creating a new database and database user
* Importing existing website data
* Updating WordPress configuration
* Configuring DNS
* Configuring Apache VirtualHost
* Securing the website with HTTPS
* Decommissioning the old DNS record

---

# 🏗️ Architecture Overview

```text
User Browser
      │
      ▼
DNS Record
(wordpress2.shyamdev.nixlabs.in)
      │
      ▼
Apache Web Server
      │
      ▼
WordPress Application
      │
      ▼
MariaDB Database
(wordpress2)
```

---

# 👤 Step 1 – Create New Linux User

Created a dedicated Linux user for the migrated website.

```bash
sudo useradd -m wordpress2
sudo passwd wordpress2
```

### ✅ Verification

```bash
id wordpress2
```

Example Output:

```text
uid=1002(wordpress2) gid=1002(wordpress2)
```

---

# 📁 Step 2 – Migrate WordPress Files

Created a new web root directory and copied all website files from the existing deployment.

```bash
sudo mkdir -p /home/wordpress2/public_html

sudo cp -a /home/wordpress1/public_html/* \
/home/wordpress2/public_html/

sudo chown -R wordpress2:wordpress2 \
/home/wordpress2
```

### ✅ Verification

```bash
ls -la /home/wordpress2/public_html
```

Example Output:

```text
wp-admin
wp-content
wp-includes
wp-config.php
index.php
...
```

---

# 🗄️ Step 3 – Create Database and Database User

Logged into MariaDB and created a new database and user.

```sql
CREATE DATABASE wordpress2;

CREATE USER 'wordpress2_user'@'localhost'
IDENTIFIED BY 'StrongPassword';

GRANT ALL PRIVILEGES ON wordpress2.* TO
'wordpress2_user'@'localhost';

FLUSH PRIVILEGES;
```

### ✅ Verification

```sql
SHOW DATABASES;
```

Example Output:

```text
wordpress2
```

---

# 📤 Step 4 – Export Existing Database

Exported the database from the old WordPress deployment.

```bash
mysqldump -u root -p wordpress1 > wordpress1.sql
```

### ✅ Verification

```bash
ls -lh wordpress1.sql
```

---

# 📥 Step 5 – Import Database

Imported the backup into the new database.

```bash
mysql -u root -p wordpress2 < wordpress1.sql
```

### ✅ Verification

```sql
USE wordpress2;
SHOW TABLES;
```

Example Output:

```text
wp_options
wp_posts
wp_users
wp_comments
...
```

---

# ⚙️ Step 6 – Update WordPress Configuration

Updated database configuration inside:

```text
/home/wordpress2/public_html/wp-config.php
```

Updated values:

```php
define('DB_NAME', 'wordpress2');
define('DB_USER', 'wordpress2_user');
define('DB_PASSWORD', 'StrongPassword');
```

---

# 🌐 Step 7 – Update Website URL

Updated WordPress URLs to point to the new domain.

```sql
UPDATE wp_options
SET option_value='https://wordpress2.shyamdev.nixlabs.in'
WHERE option_name IN ('siteurl','home');
```

### ✅ Verification

```sql
SELECT option_name, option_value
FROM wp_options
WHERE option_name IN ('siteurl','home');
```

Example Output:

```text
siteurl  https://wordpress2.shyamdev.nixlabs.in
home     https://wordpress2.shyamdev.nixlabs.in
```

---

# 🌍 Step 8 – Configure DNS

Created a DNS A Record:

```text
wordpress2.shyamdev.nixlabs.in
```

### ✅ Verification

```bash
nslookup wordpress2.shyamdev.nixlabs.in
```

Example Output:

```text
Name: wordpress2.shyamdev.nixlabs.in
Address: <SERVER_IP>
```

---

# 🔧 Step 9 – Configure Apache VirtualHost

Created a dedicated Apache VirtualHost configuration.

File:

```text
/etc/httpd/conf.d/wordpress2.conf
```

### Verify Configuration

```bash
sudo apachectl configtest
```

Example Output:

```text
Syntax OK
```

Restart Apache:

```bash
sudo systemctl restart httpd
```

---

# 🔒 Step 10 – Configure HTTPS

Generated an SSL certificate using Certbot.

```bash
sudo certbot --apache \
-d wordpress2.shyamdev.nixlabs.in
```

### Verify Certificate

```bash
sudo certbot certificates
```

Example Output:

```text
Certificate Name: wordpress2.shyamdev.nixlabs.in
Expiry Date: Valid
```

---

# 🧹 Step 11 – Remove Old DNS Record

Removed the DNS entry:

```text
wordpress1.shyamdev.nixlabs.in
```

This ensures users access the migrated website through the new domain.

---

# ✅ Validation & Testing

### Check Website Response

```bash
curl -I https://wordpress2.shyamdev.nixlabs.in
```

Expected Output:

```text
HTTP/2 200
```

### Verify Website Access

Open in browser:

```text
https://wordpress2.shyamdev.nixlabs.in
```

### Verify Admin Login

```text
https://wordpress2.shyamdev.nixlabs.in/wp-admin
```

---

# 📸 Screenshots

## Screenshot 1 – Web Server Layer

Show:

* Apache configuration test
* Website loading in browser

---

## Screenshot 2 – Database Layer

Show:

* Database created
* Database user created
* Imported WordPress tables

Commands:

```sql
SHOW DATABASES;
SHOW TABLES;
```

---

## Screenshot 3 – Application Layer

Show:

* Old website (Before Migration)
* New website (After Migration)

This demonstrates successful migration of the WordPress application.

---

# 🎯 Result

✅ WordPress files migrated successfully

✅ Database migrated successfully

✅ New database user configured

✅ DNS configured successfully

✅ Apache VirtualHost configured

✅ HTTPS enabled using Let's Encrypt

✅ Website accessible through the new domain

---

# 🌐 Live URLs

### Website

```text
https://wordpress2.shyamdev.nixlabs.in
```

### Admin Panel

```text
https://wordpress2.shyamdev.nixlabs.in/wp-admin
```
