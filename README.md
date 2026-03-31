## Flags found:
#### - FLAG{So_It_Begins.}
#### - Flag{Keep_It_Secret_Keep_It_Safe}
#### - FLAG{All_That_Is_Gold_Does_Not_Glitter}

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

## Additional Endpoints Discovered

| Endpoint | Description |
|----------|-------------|
| `/profile` | Contains a login form (username + password fields); authentication-protected |
| `/logout` | Redirects to `/profile` instead of a homepage or dedicated login page |

### Security Observations
- The `/logout` redirect behavior is atypical and may indicate weaknesses in:
  - Session handling
  - Access control logic

---

## Attack Surface Summary
The following areas warrant further testing:
- SQL injection exploitation (confirmed vulnerability)
- Authentication mechanisms at `/profile`
- Session management logic
- SSH service (if credentials are obtained)

These findings position authentication and user profile functionality as critical attack surfaces for subsequent testing phases.
