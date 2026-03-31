# Mordor-Corp-Penetration-Test-Report
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
