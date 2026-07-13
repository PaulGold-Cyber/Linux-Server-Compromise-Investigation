# Linux Server Compromise Investigation

## Overview

This project documents a complete SOC investigation into the compromise of a Linux web server. The investigation was conducted using Splunk by analyzing and correlating events across Apache access logs, Linux authentication logs, and FTP server logs.

The investigation identified a multi-stage attack originating from `172.16.20.99`, beginning with active reconnaissance and progressing through web application compromise, web shell deployment, SSH account compromise, root privilege escalation, persistence, and confirmed sensitive data exfiltration.

---

## Investigation Objectives

The primary objectives of this investigation were to:

- Identify the source of the suspicious activity.
- Reconstruct the complete attack timeline.
- Determine how the attacker gained initial access.
- Identify compromised accounts and privilege escalation activity.
- Detect persistence mechanisms established by the attacker.
- Identify sensitive files accessed or exfiltrated.
- Correlate events across multiple log sources.
- Map observed attacker behavior to the MITRE ATT&CK framework.
- Provide containment and remediation recommendations.

---

## Log Sources

| Log Source | Investigation Purpose |
|---|---|
| Apache `access.log` | Web reconnaissance, vulnerability scanning, brute-force activity, web shell execution, and post-exploitation commands |
| Linux `auth.log` | SSH authentication attempts, account compromise, sudo activity, root privilege escalation, and account creation |
| `vsftpd.log` | FTP authentication, sensitive file downloads, and suspicious file uploads |

---

## Attack Chain

`Reconnaissance` → `Vulnerability Scanning` → `Directory Enumeration` → `Web Credential Attack` → `Admin Access` → `Web Shell Deployment` → `Remote Command Execution` → `System Discovery` → `Sensitive File Discovery` → `LinPEAS Execution` → `SSH Credential Attack` → `Developer Account Compromise` → `Root Access` → `Data Exfiltration` → `Persistence`

---

## Key Findings

- **Attacker IP:** `172.16.20.99`
- **Compromised Host:** `brightstar-web`
- **Compromised Account:** `developer`
- **Persistence Account:** `svc_monitor`
- **Malicious Web Shell:** `/uploads/shell.php`
- **Transferred Enumeration Tool:** `/tmp/linpeas.sh`
- **Suspicious Uploaded Script:** `/tmp/.hidden/beacon.sh`
- **Highest Privilege Achieved:** `root`
- **Confirmed Data Exfiltration:** Yes
- **Affected Services:** HTTP, SSH, FTP

---

## Confirmed Exfiltrated Files

| File | Size | Assessment |
|---|---:|---|
| `/var/backups/customer_db.sql` | 45,102 bytes | Customer database backup |
| `/etc/ssl/private/payment_keys.pem` | 3,247 bytes | Potentially sensitive private key |
| `/var/backups/employee_data.csv` | 18,934 bytes | Employee-related data |
| `/var/www/html/config.php` | 423 bytes | Web application configuration file |

---

## Investigation Highlights

The investigation correlated activity across three independent log sources to reconstruct the attack.

One key correlation involved the file `/var/backups/customer_db.sql`. The attacker first searched for SQL files through the deployed web shell, accessed the database backup, and later successfully downloaded the same 45,102-byte file over FTP.

Another strong correlation involved the creation of the `svc_monitor` persistence account. The Apache access logs recorded the commands sent through the web shell, while the Linux authentication logs independently confirmed the creation of the account, password assignment, and addition to the `sudo` group.

The investigation also identified a timeline anomaly: FTP logs show successful authentication as `svc_monitor` at `09:02:16`, while the account creation is recorded at `09:05:01`. The available evidence does not provide a definitive explanation for this inconsistency.

---

## Skills Demonstrated

- Splunk investigation and SPL queries
- SIEM-based incident investigation
- Linux log analysis
- Web server log analysis
- SSH authentication analysis
- FTP activity analysis
- Multi-source log correlation
- Attack timeline reconstruction
- Incident triage
- IOC identification
- MITRE ATT&CK mapping
- Evidence-based analytical reasoning

---

## Project Structure

Detailed investigation documentation includes:

- Master Attack Timeline
- Web Attack Investigation
- SSH and Authentication Investigation
- FTP and Data Exfiltration Investigation
- Indicators of Compromise
- MITRE ATT&CK Mapping
- SPL Queries
- Investigation Screenshots
- Final Assessment and Remediation Recommendations

---

## Disclaimer

This project was created for educational and portfolio purposes in a controlled lab environment. All IP addresses, systems, accounts, credentials, and attack activity documented in this repository belong to the lab scenario and do not represent unauthorized activity against real-world systems.
