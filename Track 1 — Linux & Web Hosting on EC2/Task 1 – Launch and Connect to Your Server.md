# 🚀 Task 1 – Launch and Connect to Your Server

## 🎯 Objective

Deploy a Linux server on AWS, establish secure remote access, configure network access rules, and provision mentor access for validation.

---

# 📋 Task Requirements

* Launch an EC2 instance running **AlmaLinux 9**.
* Connect to the server using **SSH** and a key pair.
* Learn basic Linux file editing using **nano** or **vim**.
* Configure Security Group rules for web and administrative access.
* Create mentor access using the provided setup script.
* Verify successful configuration and server accessibility.

---

# 🏗️ Step 1: Launching the EC2 Instance

### Activities Performed

* Accessed the AWS EC2 Dashboard.
* Launched a new EC2 instance.
* Selected the official **AlmaLinux OS 9** AMI.
* Configured the instance with the required settings.
* Verified that the instance entered the **Running** state successfully.

### Why This Step Matters

An EC2 instance acts as a virtual Linux server in the cloud. This server will be used throughout the lab to deploy applications, perform system administration, and practice security configurations.

---

# 🔑 Step 2: Creating and Configuring the Key Pair

### Activities Performed

* Generated an SSH key pair during instance creation.
* Downloaded and securely stored the private key file.
* Associated the key pair with the EC2 instance.

### Why This Step Matters

SSH key pairs provide secure authentication and eliminate the need for password-based server logins.

---

# 🌐 Step 3: Configuring Network Access

### Activities Performed

Configured Security Group inbound rules to allow the required traffic:

| Service       | Port                                    | Purpose               |
| ------------- | --------------------------------------- | --------------------- |
| HTTP          | 80                                      | Web traffic           |
| HTTPS         | 443                                     | Secure web traffic    |
| Mentor Access | Configured through Security Group rules | Validation and review |

### Why This Step Matters

Security Groups function as firewalls that control who can access the server and which services are exposed.

---

# 💻 Step 4: Connecting to the Server Using SSH

### Activities Performed

* Retrieved the EC2 public IP address.
* Connected to the instance using the downloaded SSH key.
* Successfully established a secure remote terminal session.

### Why This Step Matters

SSH is the standard method used by administrators, DevOps engineers, and security professionals to manage Linux servers remotely.

---

# 🐧 Step 5: Basic Linux Administration

### Activities Performed

* Navigated the Linux environment.
* Explored system directories and files.
* Verified shell access and user permissions.

### Why This Step Matters

Linux administration skills are fundamental for cloud engineering, system administration, and cybersecurity roles.

---

# 📝 Step 6: File Editing Using a Terminal Editor

### Activities Performed

* Used a terminal-based text editor (Nano/Vim).
* Created and modified files directly from the command line.
* Saved and verified file changes.

### Why This Step Matters

Most production Linux servers do not provide a graphical interface. Configuration changes are commonly performed using command-line editors.

---

# 🤝 Step 7: Mentor Access Setup

### Activities Performed

### Downloaded the Mentor Setup Script

```bash
curl -O https://raw.githubusercontent.com/TC-Labs1/lms-scripts/refs/heads/main/linux-tc-mentor-user-creation.sh
```

### Reviewed the Script

```bash
less linux-tc-mentor-user-creation.sh
```

### Executed the Script

```bash
sudo bash linux-tc-mentor-user-creation.sh
```

### Why This Step Matters

Reviewing scripts before execution is a security best practice and helps prevent unintended changes to systems.

---

# 👤 Step 8: Mentor User Provisioning

### Activities Performed

The script automatically:

* Created the `tc-mentor` Linux user.
* Granted administrative (sudo) privileges.
* Installed the mentor's SSH public key.
* Configured mentor access for validation.

### Verification Performed

```bash
id tc-mentor
```

Verified that the mentor account was successfully created and assigned the required permissions.

### Why This Step Matters

Creating separate accounts follows security best practices by avoiding shared administrative credentials.

---

# ✅ Step 9: Validation and Testing

### Validation Performed

* Verified EC2 instance status.
* Confirmed successful SSH connectivity.
* Confirmed Linux terminal access.
* Verified file editing functionality.
* Confirmed mentor account creation.
* Verified mentor access configuration.

---

# 📸 Evidence Collected

The following screenshots were captured as proof of completion:

* Running EC2 Instance
* Successful SSH Session
* Security Group Configuration
* Mentor User Verification

---

# 🎓 Key Concepts Learned

### AWS EC2

Learned how to provision and manage a cloud-based Linux server.

### SSH Authentication

Learned secure remote administration using SSH key pairs.

### Security Groups

Learned how network-level access control protects cloud resources.

### Linux Administration

Learned basic command-line operations and server management.

### User Management

Learned how Linux users, permissions, and sudo access are configured.

### Security Best Practices

Learned the importance of reviewing scripts before execution and using separate accounts for verification activities.

---

# 🏁 Outcome

Successfully deployed an AlmaLinux 9 EC2 instance, established secure remote access, configured network access controls, performed Linux administration tasks, and provisioned mentor access for verification.

**Status:** ✅ Completed Successfully
