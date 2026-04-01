## Flags found:
#### - FLAG{So_It_Begins.}
#### - Flag{Keep_It_Secret_Keep_It_Safe}
#### - FLAG{All_That_Is_Gold_Does_Not_Glitter}
#### - FLAG{Mithril_Master}

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

### Exposed Resources

A publicly accessible internal chat log was discovered at `/uploads/chat.log` containing:
- Administrator-user conversation discussing hidden development artifacts
- Suggestion to examine asset files thoroughly

### Enumeration Opportunities
The chat log reference suggests potential sensitive information may reside in:
- JavaScript files
- CSS files  
- Images
- Other static resources

This increases the likelihood of discovering hidden endpoints, credentials, or developer notes during further enumeration.

# Enumeration & Mapping

## Open Ports & Services
An Nmap scan revealed two open TCP services:

| Port | Service | Version |
|------|---------|---------|
| 22 | OpenSSH | 8.4p1 (Debian 11) |
| 80 | Apache HTTP Server | 2.4.54 (Debian) |

### Service Analysis
- **SSH (port 22):** Potential attack surface if weak credentials, reused passwords, exposed SSH keys, or privilege escalation vulnerabilities exist.
- **HTTP (port 80):** Hosts the Mordor Corp website—likely the primary attack surface.

### Operating System
The host runs a Linux-based OS. Nmap OS fingerprinting suggests Linux kernel versions between 4.x and 5.x.

### Exposed Services Summary
No additional common services (MySQL, FTP, SMB, HTTPS) are exposed externally, suggesting entry points are limited to:
- SSH access
- The web application

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

#### `users` table contents:
| id | username | email | role | password (hash) |
|----|----------|-------|------|-----------------|
| 1 | saruman | saruman@mordor-corp.fi | user | `6ca3e14cf5ffd2108aa869f4c16394ad` (MD5) |
| 2 | sauron | sauron@mordor-corp.fi | admin | `$2y$14$SWEbY90.fjIbMTvrPHz4teoesnRaWyrO9LPB8NQijFIy4rckSmHnm` (bcrypt) |

---

## Additional Endpoints Discovered

| Endpoint | Description |
|----------|-------------|
| `/profile` | Contains a login form (username + password fields); authentication-protected. Also vulnerable to time‑based blind SQL injection. |
| `/logout` | Redirects to `/profile` instead of a homepage or dedicated login page. |
| `/upload` | File upload interface – requires admin access. |
| `/uploads` | Directory listing (exposed) containing `chat.log` and `.ssh/sauron_rsa`. |
| `/assets` | Static assets directory – contains images, CSS, JS, and a flag (see below). |
| `/server-status` | Apache status page (leaks internal IPs, software versions, and active requests). |

### Exposed SSH Private Key
A private SSH key was discovered at `/uploads/.ssh/sauron_rsa`:

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn...
-----END OPENSSH PRIVATE KEY-----

This key belongs to user `sauron` and is publicly accessible – a critical misconfiguration that provides direct system access.

### Apache Server-Status Leak
The `/server-status` endpoint (accessible without authentication) revealed:
- **Software:** Apache/2.4.54, PHP/7.4.33
- **Internal IP addresses:** `172.19.0.2`, `172.19.0.1`
- **Suspicious request paths:** `/zorum`, `/zh-tw`, `/zh-cn`, `/zoom`, `/zone`, `/zope`, `/zh_TW`, `/zones` – these may indicate hidden admin panels or test pages.

### Assets Directory Analysis
The `/assets/` directory was recursively downloaded and examined:
- `script.js` – minimal JavaScript (no secrets)
- `style.css` – standard styles (no secrets)
- `Mordor.png` – image containing no hidden metadata or steganographic content
- No additional sensitive files were found.

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
- Known vulnerabilities affecting Apache 2.4.54 or OpenSSH 8.4p1
- Forgotten legacy functionality or old database tables (e.g., `moria`)

These findings position authentication, SSH access, and the file upload functionality as critical attack surfaces for subsequent testing phases.

# Initial Access

## Attempt 1: SSH with Exposed Private Key
**Vulnerability tested:** Exposed SSH private key (misconfiguration)  
**Method:** Direct SSH login using the discovered key  
**Command:** `ssh -i sauron_rsa sauron@mordor-corp.fi`  
**Outcome:** ✅ **Success** – The key was not passphrase-protected and provided immediate access to the system as user `sauron`.

After connecting, the user flag was found in `~/user.txt`:


## Attempt 2: Cracking Saruman's MD5 Hash
**Vulnerability tested:** Weak password hash (MD5)  
**Method:** Dictionary attack with rockyou.txt using hashcat  
**Command:** `hashcat -m 0 -a 0 saruman_hash.txt /usr/share/wordlists/rockyou.txt`  
**Outcome:** ❌ **Failed** – The hash was not present in the rockyou wordlist. No further attempts were made as SSH access was already obtained.

## Attempt 3: SQL Injection Bypass on `/profile/`
**Vulnerability tested:** Time‑based blind SQL injection on login form  
**Method:** Attempted simple bypass payload  
**Payload:** `username=admin' OR '1'='1' -- &password=anything`  
**Outcome:** ❌ **Failed** – The form did not authenticate with this payload. Since SSH access was already achieved, no deeper exploitation was pursued.

## Summary of Initial Access
- **Successful vector:** SSH private key exposure.
- **Credentials obtained:** None needed – key‑based authentication.
- **Privilege level:** `sauron` (admin role in web context, but standard user in the OS).

# Privilege Escalation

## Initial System Enumeration (as `sauron`)
- **OS:** Debian 11 (bullseye)
- **Kernel:** 5.10.0
- **Sudo rights:** `sauron` may have `sudo` privileges or other escalation vectors.
- **Running processes:** Web server (Apache), MySQL, SSH.


