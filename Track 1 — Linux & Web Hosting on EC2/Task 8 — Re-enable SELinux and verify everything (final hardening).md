# Task 8 - Re-enable SELinux and Verify Everything (Final Hardening)

## Objective

Re-enable SELinux in **Enforcing** mode, ensure the configuration persists across reboots, resolve any SELinux-related issues using the correct policies, and verify that all production websites remain accessible over HTTPS.

---

## Environment

| Component | Status |
|-----------|--------|
| Operating System | AlmaLinux 9 |
| Web Server | Apache HTTP Server |
| Application Server | Gunicorn |
| Database | MariaDB |
| SELinux Mode | Enforcing |

---

# Changes Performed

## 1. Enabled SELinux Enforcing Mode

Verified the current SELinux status:

```bash
getenforce
```

Output:

```text
Enforcing
```

Verified persistent configuration:

```bash
grep ^SELINUX= /etc/selinux/config
```

Output:

```text
SELINUX=enforcing
```

This ensures SELinux remains in **Enforcing** mode even after a system reboot.

---

## 2. Verified Apache Reverse Proxy Communication

### Issue

When SELinux is enabled, Apache may be prevented from communicating with backend services such as Gunicorn over local TCP ports.

### Resolution

Verified the required SELinux boolean was enabled to allow Apache network connections.

```bash
sudo setsebool -P httpd_can_network_connect 1
```

### Result

Apache successfully communicated with the Gunicorn backend, and the Django application remained accessible over HTTPS.

---

## 3. Verified File Security Contexts

### Issue

SELinux requires web content to have the correct security context before Apache can access files.

### Resolution

Verified and restored SELinux file contexts where necessary.

Example:

```bash
sudo restorecon -Rv /var/www
```

### Result

Apache successfully accessed all website files while SELinux remained in Enforcing mode.

---

## 4. Verified SSL Certificate Access

### Issue

SELinux may block Apache from accessing SSL certificates if file contexts are incorrect.

### Resolution

Verified the certificate directory and restored default contexts where required.

```bash
sudo restorecon -Rv /etc/letsencrypt
```

### Result

HTTPS certificates remained accessible and all websites continued serving encrypted traffic successfully.

---

## 5. Checked for SELinux Denials

Executed:

```bash
sudo ausearch -m avc -ts recent
```

Output:

```text
<no matches>
```

### Result

No SELinux Access Vector Cache (AVC) denials were detected after enabling Enforcing mode.

---

# Website Verification

## Static Website

```bash
curl -I https://static1.shyamdev.nixlabs.in
```

Result:

```text
HTTP/1.1 200 OK
```

---

## WordPress Website

```bash
curl -I https://wordpress2.shyamdev.nixlabs.in
```

Result:

```text
HTTP/1.1 200 OK
```

---

## Django Website

```bash
curl -I https://django2.shyamdev.nixlabs.in
```

Result:

```text
HTTP/1.1 200 OK
```

---

# Verification Summary

| Verification | Result |
|--------------|--------|
| SELinux Mode | ✅ Enforcing |
| Persistent After Reboot | ✅ Yes |
| AVC Denials | ✅ None |
| Static Website | ✅ HTTP 200 |
| WordPress Website | ✅ HTTP 200 |
| Django Website | ✅ HTTP 200 |
| HTTPS Enabled | ✅ Yes |
| Used `setenforce 0` | ❌ No |
| Used Permissive Mode | ❌ No |

---

# Issues Encountered

During validation after enabling **SELinux Enforcing**, **no service failures or SELinux access denials were encountered**.

The following verification steps were completed:

- Verified SELinux was running in **Enforcing** mode.
- Confirmed there were **no AVC denials** (`ausearch -m avc -ts recent` returned `<no matches>`).
- Confirmed Apache successfully served the static website.
- Confirmed Apache successfully served the WordPress website.
- Confirmed Apache successfully proxied requests to the Django application.
- Confirmed all websites were accessible over HTTPS and returned **HTTP 200 OK**.

Since no SELinux policy violations occurred after enabling Enforcing mode, **no additional SELinux policy modifications were required** beyond ensuring the correct configuration was already in place.

---

# Conclusion

SELinux was successfully re-enabled and configured to operate in **Enforcing** mode. The configuration is persistent across system reboots, and no permissive mode or temporary SELinux disablement was used.

All production websites remained fully operational over HTTPS, returned **HTTP 200 OK**, and no SELinux AVC denials were detected during validation. This confirms that the system is operating in a secure, production-ready state with SELinux enforcing the appropriate security policies.
