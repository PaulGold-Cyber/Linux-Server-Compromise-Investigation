# Linux Server Compromise Investigation

## Overview

This project documents a complete SOC investigation into the compromise of a Linux web server using Splunk.

The investigation was conducted by analyzing and correlating events across three primary log sources:

- Apache web access logs
- Linux authentication logs
- FTP server logs

The investigation identified a multi-stage attack originating from `172.16.20.99`, progressing from active reconnaissance and credential attacks to web application compromise, PHP web shell deployment, SSH account compromise, root privilege escalation, persistence, and confirmed sensitive data exfiltration.

**Final Incident Severity: Critical**

---

## Attack Chain

```text
Reconnaissance
      ↓
Web Vulnerability Scanning
      ↓
Directory Enumeration
      ↓
Credential Attacks
      ↓
Administrative Web Access
      ↓
PHP Web Shell Deployment
      ↓
Remote Command Execution
      ↓
System and File Discovery
      ↓
SSH Account Compromise
      ↓
Root Privilege Escalation
      ↓
Persistence
      ↓
Sensitive Data Collection
      ↓
Confirmed FTP Data Exfiltration
```

---

## Key Findings

The investigation confirmed the following malicious activity:

- Active reconnaissance using **Nmap**.
- Web vulnerability scanning using **Nikto**.
- Hidden directory discovery using **Gobuster**.
- Credential attacks using **Hydra**.
- Successful administrative web access.
- Deployment and execution of a PHP web shell.
- Remote Linux command execution through `shell.php`.
- Transfer and execution of **LinPEAS** for privilege escalation enumeration.
- Successful SSH compromise of the legitimate `developer` account.
- Access to `/etc/shadow` using elevated privileges.
- Successful escalation to an interactive root shell.
- Creation of the privileged persistence account `svc_monitor`.
- Successful FTP authentication as `root`.
- Confirmed exfiltration of multiple sensitive files.
- Upload of the suspicious hidden script `/tmp/.hidden/beacon.sh`.

---

## Confirmed Data Exfiltration

The following sensitive files were successfully downloaded from the compromised server:

| File | Assessment |
|---|---|
| `/var/backups/customer_db.sql` | Customer database backup |
| `/etc/ssl/private/payment_keys.pem` | Potentially sensitive private key material |
| `/var/backups/employee_data.csv` | Employee-related information |
| `/var/www/html/config.php` | Web application configuration |

The investigation also confirmed privileged access to:

```text
/etc/shadow
```

This exposed local password hashes and introduced additional credential-compromise risk.

---

## Investigation Methodology

Because the available dataset contained a relatively small number of events, the investigation began with a complete chronological review of the logs.

After suspicious activity was identified, targeted Splunk searches were used to isolate:

- The attacker IP address
- Offensive security tools
- Web shell activity
- Authentication failures and successful logins
- Privilege escalation
- Persistence mechanisms
- Sensitive file access
- FTP uploads and downloads

Events were then correlated across Apache, Linux authentication, and FTP logs to reconstruct the complete attack timeline.

The investigation followed this workflow:

```text
Full Log Review
      ↓
Identify Suspicious Activity
      ↓
Isolate Attacker Infrastructure
      ↓
Search Accounts, Files and Tools
      ↓
Correlate Multiple Log Sources
      ↓
Reconstruct Attack Timeline
      ↓
Extract IOCs
      ↓
Map Behaviors to MITRE ATT&CK
      ↓
Assess Impact and Recommend Remediation
```

---

## Splunk Investigation & Evidence

The Splunk section contains the SPL queries used to investigate the incident, targeted searches used to validate findings, detection examples developed after the investigation, and screenshots of actual Splunk results.

**View the complete Splunk investigation:**

[Splunk SPL Queries and Evidence](splunk/spl-queries.md)

The investigation included searches for:

- Complete chronological dataset review
- Attacker IP activity
- Apache web activity
- Offensive security tools
- PHP web shell execution
- Sensitive file access
- `developer` account compromise
- SSH authentication activity
- Root privilege escalation
- `svc_monitor` persistence
- FTP file transfers
- Confirmed data exfiltration
- Cross-source database correlation

---

## Investigation Report

The complete investigation is divided into seven sections:

| Section | Description |
|---|---|
| [01 - Master Attack Timeline](investigation/01-master-timeline.md) | Complete chronological reconstruction of the attack |
| [02 - Web Investigation](investigation/02-web-investigation.md) | Reconnaissance, scanning, credential attacks, web shell deployment and command execution |
| [03 - Authentication Investigation](investigation/03-authentication-investigation.md) | SSH compromise, sudo abuse, root privilege escalation and persistence |
| [04 - FTP and Data Exfiltration](investigation/04-ftp-and-exfiltration.md) | FTP authentication, sensitive file downloads and suspicious uploads |
| [05 - Indicators of Compromise](investigation/05-iocs.md) | Attacker IPs, accounts, files, tools and other observable indicators |
| [06 - MITRE ATT&CK Mapping](investigation/06-mitre-attack.md) | Mapping observed attacker behaviors to MITRE ATT&CK techniques |
| [07 - Conclusion and Recommendations](investigation/07-conclusion-and-recommendations.md) | Final incident assessment, impact, containment and remediation recommendations |

---

## Indicators of Compromise

### Network Indicator

```text
172.16.20.99
```

### Compromised Account

```text
developer
```

### Attacker-Created Persistence Account

```text
svc_monitor
```

### Malicious and Suspicious Files

```text
/uploads/shell.php
/tmp/linpeas.sh
/tmp/.hidden/beacon.sh
```

### Offensive Tools Observed

```text
Nmap
Nikto
Gobuster
Hydra
LinPEAS
```

For the complete IOC analysis, see:

[Indicators of Compromise](investigation/05-iocs.md)

---

## MITRE ATT&CK

Observed attacker behaviors were mapped to the MITRE ATT&CK framework, including techniques associated with:

- Active scanning
- Brute-force credential attacks
- Web shell deployment
- Command and scripting execution
- System and file discovery
- Valid account abuse
- Sudo and root privilege escalation
- Local account creation
- Sensitive data collection
- Exfiltration over network services

View the complete mapping:

[MITRE ATT&CK Mapping](investigation/06-mitre-attack.md)

---

## Skills Demonstrated

This project demonstrates practical experience with:

- Splunk investigation
- SPL searching and filtering
- Linux log analysis
- Apache access log analysis
- SSH authentication analysis
- FTP log analysis
- Incident investigation
- Timeline reconstruction
- Cross-source event correlation
- Brute-force detection
- Web shell identification
- Privilege escalation analysis
- Persistence identification
- Data exfiltration analysis
- IOC extraction
- MITRE ATT&CK mapping
- Incident severity assessment
- Containment and remediation planning
- Technical security documentation

---

## Tools and Technologies

- **Splunk Enterprise**
- **SPL**
- **Linux**
- **Apache**
- **SSH**
- **FTP**
- **MITRE ATT&CK**
- **GitHub**
- **Markdown**

---

## Analytical Limitations

The investigation was based primarily on:

- Apache `access.log`
- Linux `auth.log`
- FTP `vsftpd.log`

Several limitations remain:

- The contents of `beacon.sh` were unavailable.
- No file hashes were available for suspicious files.
- No packet capture or detailed network telemetry was provided.
- The exact contents of exfiltrated files were unavailable.
- The original source of the successful `developer` credentials remains unknown.
- The source of the successful root FTP credentials remains unknown.
- A timestamp inconsistency exists involving `svc_monitor`.
- Because the attacker obtained root-level access, additional activity may have occurred outside the visibility of the provided logs.

---

## Final Assessment

The investigation confirmed a complete compromise of the Linux server `brightstar-web`.

The attacker successfully progressed from reconnaissance and credential attacks to web application compromise, remote command execution, SSH account compromise, root privilege escalation, persistence, and confirmed sensitive data exfiltration.

The affected host should be considered fully compromised and untrusted.

**Final Severity: Critical**

The recommended response includes immediate network isolation, forensic evidence preservation, credential and secret rotation, investigation of the broader environment, and rebuilding the affected server from a known-good trusted image.

---

## Repository Structure

```text
Linux-Server-Compromise-Investigation/
│
├── README.md
│
├── investigation/
│   ├── 01-master-timeline.md
│   ├── 02-web-investigation.md
│   ├── 03-authentication-investigation.md
│   ├── 04-ftp-and-exfiltration.md
│   ├── 05-iocs.md
│   ├── 06-mitre-attack.md
│   └── 07-conclusion-and-recommendations.md
│
├── splunk/
│   └── spl-queries.md
│
└── screenshots/
    ├── 01-initial-dataset-review.png.png
    ├── 02-attacker-ip-activity.png
    ├── 03-apache-web-activity.png
    ├── 04-offensive-security-tools.png
    ├── 05-web-shell-activity.png
    ├── 06-sensitive-file-activity.png
    ├── 07-developer-account-investigation.png
    ├── 08-ssh-authentication.png
    ├── 09-root-privilege-escalation.png
    ├── 10-svc-monitor-persistence.png
    ├── 11-ftp-file-transfers.png
    ├── 12-confirmed-data-exfiltration.png
    └── 13-customer-database-correlation.png
```

---

## Disclaimer

This project was conducted in a controlled lab environment for educational and portfolio purposes. All IP addresses, systems, accounts, and attack activity documented in this repository are part of the simulated investigation environment.
