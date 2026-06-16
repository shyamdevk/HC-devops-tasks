# Task 2 - Install Apache Web Server and Verify PHP

## Objective

Install Apache (httpd) and PHP on an AlmaLinux 9 server, configure the required PHP modules, and verify that PHP is functioning correctly through a web browser.

---

## Environment

| Component        | Value           |
| ---------------- | --------------- |
| Operating System | AlmaLinux 9     |
| Web Server       | Apache (httpd)  |
| PHP Version      | PHP 8.x         |
| Document Root    | `/var/www/html` |

---

## Requirements

The following PHP modules were installed:

* mysqli
* gd
* xml
* mbstring
* json
* curl

---

## Implementation Steps

### 1. Update System Packages

```bash
sudo dnf update -y
```

### 2. Install Apache Web Server

```bash
sudo dnf install httpd -y
```

### 3. Start and Enable Apache

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

Verify service status:

```bash
sudo systemctl status httpd
```

Expected output:

```text
active (running)
```

---

### 4. Install PHP and Required Modules

```bash
sudo dnf install php php-mysqlnd php-gd php-xml php-mbstring php-json php-curl -y
```

Verify PHP installation:

```bash
php -v
```

---

### 5. Restart Apache

```bash
sudo systemctl restart httpd
```

---

### 6. Create PHP Verification Page

Create the file:

```bash
sudo nano /var/www/html/phpinfo.php
```

Add the following content:

```php
<?php
phpinfo();
?>
```

Save and exit.

---

### 7. Configure Firewall

Allow HTTP traffic:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-services
```

Expected output should contain:

```text
http
```

---

## Verification

### PHP Information Page

The PHP information page is accessible at:

**Live URL:**

http://100.53.9.161/phpinfo.php

Or:

[View PHP Info Page](http://100.53.9.161/phpinfo.php)

---

### Required PHP Modules Verified

| Module   | Status |
| -------- | ------ |
| mysqli   | Loaded |
| gd       | Loaded |
| xml      | Loaded |
| mbstring | Loaded |
| json     | Loaded |
| curl     | Loaded |

---

### Apache Auto-Start Verification

Reboot the server:

```bash
sudo reboot
```

Reconnect and verify:

```bash
sudo systemctl status httpd
```

Result:

```text
active (running)
```

This confirms Apache automatically starts after a reboot.

---

## Evidence

### Screenshot 1

PHP Information page loaded successfully in browser.

### Screenshot 2

Required PHP modules visible in the PHP Information page.

### Screenshot 3

Apache service status showing:

```text
active (running)
```

after server reboot.

---

## Outcome

Successfully installed Apache Web Server and PHP on AlmaLinux 9.

Verified:

* Apache service is running
* PHP is functioning correctly
* Required PHP modules are loaded
* PHP information page is accessible through a browser
* Apache automatically starts after system reboot

Task completed successfully.
