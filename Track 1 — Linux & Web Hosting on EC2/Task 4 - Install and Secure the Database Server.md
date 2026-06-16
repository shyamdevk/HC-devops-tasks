# 🗄️ Task 4 - Install and Secure the Database Server

## 📋 Objective

Install and secure a MariaDB database server, then provision isolated databases and users for two web applications:

* WordPress
* Django

Each application must have:

* Its own dedicated database
* Its own dedicated database user
* A strong password
* Access only to its own database

---

## 🖥️ Environment

| Component       | Value       |
| --------------- | ----------- |
| OS              | AlmaLinux 9 |
| Database Server | MariaDB     |
| Access Method   | SSH         |
| Database Client | MariaDB CLI |

---

# 🚀 MariaDB Installation

Update system packages:

```bash
sudo dnf update -y
```

Install MariaDB server:

```bash
sudo dnf install mariadb-server -y
```

Start MariaDB:

```bash
sudo systemctl start mariadb
```

Enable automatic startup on boot:

```bash
sudo systemctl enable mariadb
```

Verify service status:

```bash
sudo systemctl status mariadb
```

Expected result:

```text
active (running)
```

---

# 🔐 Secure MariaDB Installation

Run the security configuration wizard:

```bash
sudo mysql_secure_installation
```

Security actions performed:

* Set root password
* Removed anonymous users
* Disabled remote root login
* Removed test database
* Reloaded privilege tables

---

# 👤 Database Provisioning

## WordPress Database

### Create Database

```sql
CREATE DATABASE wordpress1;
```

### Create User

```sql
CREATE USER 'wordpress1_user'@'localhost'
IDENTIFIED BY '********';
```

### Grant Permissions

```sql
GRANT ALL PRIVILEGES
ON wordpress1.*
TO 'wordpress1_user'@'localhost';
```

---

## Django Database

### Create Database

```sql
CREATE DATABASE django1;
```

### Create User

```sql
CREATE USER 'django1_user'@'localhost'
IDENTIFIED BY '********';
```

### Grant Permissions

```sql
GRANT ALL PRIVILEGES
ON django1.*
TO 'django1_user'@'localhost';
```

---

## Apply Changes

```sql
FLUSH PRIVILEGES;
```

---

# 🔍 Security Verification

## Verify MariaDB Users

```sql
SELECT User, Host FROM mysql.user;
```

Example output:

```text
+-----------------+-----------+
| User            | Host      |
+-----------------+-----------+
| django1_user    | localhost |
| mariadb.sys     | localhost |
| mysql           | localhost |
| root            | localhost |
| wordpress1_user | localhost |
+-----------------+-----------+
```

### Validation

* Root restricted to localhost
* No anonymous users present
* Application users created successfully

---

# 🧪 Database Access Verification

## WordPress User

Login:

```bash
mysql -u wordpress1_user -p
```

Check accessible databases:

```sql
SHOW DATABASES;
```

Output:

```text
information_schema
wordpress1
```

Confirmed:

* Can access `wordpress1`
* Cannot access `django1`

---

## Django User

Login:

```bash
mysql -u django1_user -p
```

Check accessible databases:

```sql
SHOW DATABASES;
```

Output:

```text
information_schema
django1
```

Confirmed:

* Can access `django1`
* Cannot access `wordpress1`

---

# 📊 Databases Created

| Application | Database   | User            |
| ----------- | ---------- | --------------- |
| WordPress   | wordpress1 | wordpress1_user |
| Django      | django1    | django1_user    |

---

# ✅ Definition of Done

* MariaDB installed successfully
* MariaDB service running
* MariaDB enabled at boot
* Root password configured
* Remote root login disabled
* Anonymous users removed
* Test database removed
* Dedicated WordPress database created
* Dedicated Django database created
* Dedicated database users created
* Least-privilege access implemented
* Users can access only their assigned databases

---

# 🎯 Result

A secure MariaDB server was deployed and configured following database isolation and least-privilege principles. Separate databases and users were provisioned for WordPress and Django, ensuring that each application can access only its own data while preventing cross-application access.
