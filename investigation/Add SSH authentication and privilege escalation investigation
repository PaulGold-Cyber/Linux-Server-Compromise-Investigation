# FTP and Data Exfiltration Investigation

## Overview

This section documents the investigation of malicious FTP activity associated with the compromise of the Linux server `brightstar-web`.

Analysis of `vsftpd.log` identified early FTP reconnaissance attempts, successful authentication as `root`, confirmed exfiltration of multiple sensitive files, the upload of a suspicious shell script into a hidden directory, and additional activity involving the persistence account `svc_monitor`.

The malicious FTP activity originated from `172.16.20.99`, the same IP address associated with the earlier web reconnaissance, credential attacks, PHP web shell activity, SSH account compromise, and root privilege escalation.

---

## 1. Early FTP Reconnaissance

At approximately `08:25`, the attacker began testing the exposed FTP service.

The following authentication attempts were observed:

`08:25:21` — Connection from `172.16.20.99`

`08:25:22` — Failed login attempt as `admin`

`08:25:24` — Connection from `172.16.20.99`

`08:25:25` — Failed login attempt as `root`

### Analyst Assessment

The attacker tested high-value account names against the FTP service during the early reconnaissance phase of the incident.

This activity occurred during the same general timeframe as the Nmap-based reconnaissance against the web server, demonstrating that the attacker was investigating multiple exposed services rather than focusing exclusively on HTTP.

**Confidence:** High

---

## 2. Successful FTP Authentication as `root`

At `09:00:01`, a new FTP connection was established from:

`172.16.20.99`

At `09:00:02`, the following event was recorded:

`[root] OK LOGIN`

### Cross-Source Correlation

The successful FTP authentication occurred shortly after the compromise of the `developer` account and successful privilege escalation to root:

| Timestamp | Event |
|---|---|
| `08:55:01` | Successful SSH authentication as `developer` |
| `08:55:10` | `/etc/shadow` accessed using sudo |
| `08:56:02` | Interactive root shell obtained |
| `08:58:05` | Second root session opened |
| `09:00:02` | Successful FTP authentication as `root` |

### Analyst Assessment

The logs directly confirm that the attacker successfully authenticated to the FTP service as `root`.

However, the available evidence does not conclusively demonstrate how the attacker obtained the credentials required for the successful root FTP login.

Possible sources could include credential access following the compromise, previously exposed credentials, or another unknown method. No single explanation can be confirmed from the available logs.

**Confidence:** Confirmed for successful root FTP authentication; unconfirmed regarding the source of the credentials.

---

## 3. Customer Database Exfiltration

At `09:00:05`, the attacker successfully downloaded:

`/var/backups/customer_db.sql`

File size:

`45,102 bytes`

The FTP server recorded:

`OK DOWNLOAD`

### Cross-Source Correlation

Earlier in the attack, the Apache access logs showed the following sequence:

1. The attacker searched the filesystem for SQL files:

   `find / -name *.sql -type f`

2. The attacker requested:

   `cat /var/backups/customer_db.sql`

3. The corresponding HTTP response size was:

   `45,102 bytes`

4. The same file was later successfully downloaded through FTP with an identical size of:

   `45,102 bytes`

### Analyst Assessment

This represents one of the strongest correlations in the investigation.

The evidence demonstrates the progression:

**Discovery → Access → Exfiltration**

The identical file size across the Apache and FTP evidence provides additional support that the same database backup identified during web shell activity was later successfully exfiltrated.

**Confidence:** Confirmed

---

## 4. Private Key Exfiltration

At `09:00:18`, the attacker successfully downloaded:

`/etc/ssl/private/payment_keys.pem`

File size:

`3,247 bytes`

The FTP server recorded:

`OK DOWNLOAD`

### Analyst Assessment

The filename and path indicate that the file may contain sensitive private key material associated with a payment-related system.

However, because the actual file contents were not available for analysis, the precise purpose and sensitivity of the key cannot be independently verified.

The successful download itself is directly confirmed by the FTP logs.

**Confidence:** Confirmed download; exact purpose of the key file unverified.

---

## 5. Employee Data Exfiltration

At `09:00:25`, the attacker successfully downloaded:

`/var/backups/employee_data.csv`

File size:

`18,934 bytes`

The FTP server recorded:

`OK DOWNLOAD`

### Analyst Assessment

The filename strongly suggests that the file contains employee-related information.

Because the actual file contents were not available, the specific data fields and level of sensitivity cannot be independently determined.

Nevertheless, the successful exfiltration of the file is directly confirmed by the FTP logs.

**Confidence:** Confirmed download; exact contents unverified.

---

## 6. Web Application Configuration Exfiltration

At `09:00:32`, the attacker successfully downloaded:

`/var/www/html/config.php`

File size:

`423 bytes`

The FTP server recorded:

`OK DOWNLOAD`

### Analyst Assessment

Web application configuration files may contain sensitive information such as:

- Database connection strings
- Usernames and passwords
- API keys
- Tokens
- Internal hostnames
- Application secrets

However, the actual contents of `config.php` were not available for analysis, so the presence of any specific credentials or secrets cannot be confirmed.

**Confidence:** Confirmed download; exact contents unverified.

---

## 7. Confirmed Exfiltration Summary

The following files were successfully downloaded from the compromised server:

| Timestamp | File | Size | Assessment |
|---|---|---:|---|
| `09:00:05` | `/var/backups/customer_db.sql` | 45,102 bytes | Customer database backup |
| `09:00:18` | `/etc/ssl/private/payment_keys.pem` | 3,247 bytes | Potentially sensitive private key |
| `09:00:25` | `/var/backups/employee_data.csv` | 18,934 bytes | Employee-related data |
| `09:00:32` | `/var/www/html/config.php` | 423 bytes | Web application configuration |

### Analyst Assessment

Unlike earlier HTTP events where a successful HTTP response did not necessarily prove successful retrieval of underlying file contents, the FTP logs explicitly recorded `OK DOWNLOAD`.

Therefore, successful data exfiltration is directly confirmed.

**Confidence:** Confirmed

---

## 8. Suspicious Upload of `beacon.sh`

At `09:00:40`, the attacker successfully uploaded:

`/tmp/.hidden/beacon.sh`

File size:

`1,823 bytes`

The FTP server recorded:

`OK UPLOAD`

### Analyst Assessment

Several characteristics make this file suspicious:

- The filename `beacon.sh` suggests possible periodic communication or beaconing functionality.
- The file was stored inside a hidden directory named `.hidden`.
- The upload occurred during an active compromise after root access and sensitive data exfiltration.

Possible purposes could include:

- Command-and-control communication
- Persistence
- Reverse shell functionality
- Periodic callback activity
- Additional post-exploitation automation

However, the actual script contents were not available, and the provided evidence does not confirm execution of the script or successful network communication.

Therefore, the investigation does not claim that command-and-control beaconing was definitively established.

**Confidence:** Confirmed upload; functionality and execution unconfirmed.

---

## 9. FTP Activity Using `svc_monitor`

At `09:02:16`, successful FTP authentication was observed using the account:

`svc_monitor`

The source IP remained:

`172.16.20.99`

The account subsequently downloaded:

`09:02:20` — `/var/log/auth.log`

`09:02:28` — `/var/log/apache2/access.log`

### Analyst Assessment

These two files contain evidence of significant portions of the attack:

- `auth.log` contains SSH authentication attempts, the successful `developer` login, sudo activity, and root privilege escalation.
- `access.log` contains web reconnaissance, credential attacks, web shell activity, and post-exploitation commands.

The exact purpose of downloading these logs cannot be conclusively determined.

Possible motivations could include reviewing evidence of the intrusion, identifying which attacker actions were logged, preparing for later log tampering, or collecting additional system information.

None of these motivations can be confirmed from the available evidence.

**Confidence:** Confirmed downloads; attacker motivation unconfirmed.

---

## 10. Timeline Anomaly Involving `svc_monitor`

Correlation with the Linux authentication logs identified a significant timestamp inconsistency:

| Timestamp | Event |
|---|---|
| `09:02:16` | Successful FTP authentication as `svc_monitor` |
| `09:02:20` | Download of `/var/log/auth.log` |
| `09:02:28` | Download of `/var/log/apache2/access.log` |
| `09:05:01` | `svc_monitor` recorded as newly created |
| `09:05:08` | Password assigned to `svc_monitor` |
| `09:05:15` | `svc_monitor` added to the `sudo` group |

According to the available timestamps, the account authenticated successfully approximately three minutes before its recorded creation.

### Analyst Assessment

The available evidence does not provide a definitive explanation for this inconsistency.

Possible causes could include:

- Timestamp differences between log sources.
- Missing earlier account creation events.
- Account recreation.
- Limitations or inconsistencies in synthetic lab data.

These remain hypotheses only and cannot be confirmed from the available evidence.

The anomaly is therefore documented without forcing an unsupported conclusion.

---

## FTP Attack Progression

The FTP investigation established the following sequence:

**Early FTP Probing → Root Privilege Escalation Through SSH → Successful FTP Root Login → Sensitive Data Exfiltration → Suspicious Script Upload → `svc_monitor` FTP Activity → Authentication Log Downloads**

---

## Key FTP Investigation Findings

- The attacker tested the FTP service early in the attack using `admin` and `root`.
- Successful FTP authentication as `root` was later confirmed.
- Four sensitive files were successfully exfiltrated.
- The customer database backup was correlated across Apache and FTP logs.
- A suspicious script named `beacon.sh` was uploaded to a hidden directory.
- The evidence does not confirm execution or C2 communication associated with `beacon.sh`.
- The `svc_monitor` account successfully authenticated to FTP and downloaded two important log files.
- A significant timestamp inconsistency involving `svc_monitor` was identified and documented.

---

## Splunk Investigation Evidence

The following evidence should be supported by actual Splunk screenshots and SPL queries:

1. Early failed FTP login attempts as `admin` and `root`.
2. Successful FTP authentication as `root`.
3. Download of `/var/backups/customer_db.sql`.
4. Correlation of the `45,102`-byte database file across Apache and FTP logs.
5. Download of `/etc/ssl/private/payment_keys.pem`.
6. Download of `/var/backups/employee_data.csv`.
7. Download of `/var/www/html/config.php`.
8. Upload of `/tmp/.hidden/beacon.sh`.
9. Successful FTP authentication as `svc_monitor`.
10. Downloads of `auth.log` and `access.log`.
11. The timestamp anomaly involving `svc_monitor`.

> Screenshots and SPL queries should reflect the actual investigation performed in Splunk and should not be fabricated solely for presentation.

---

## Final Assessment

The FTP logs provide direct evidence of successful sensitive data exfiltration from the compromised Linux server.

Following the compromise of the `developer` account and successful privilege escalation to root, the attacker authenticated to the FTP service as `root` and downloaded multiple sensitive files, including a customer database backup, a potentially sensitive private key, employee-related data, and the web application's configuration file.

The attacker subsequently uploaded a suspicious script named `beacon.sh` into a hidden directory. Although the filename and location suggest possible beaconing, persistence, or command-and-control functionality, neither execution nor network communication was confirmed by the available evidence.

Additional FTP activity involving the `svc_monitor` account included successful downloads of the Linux authentication log and Apache access log. A timestamp inconsistency was identified because this activity occurred before the account's recorded creation time.

The FTP evidence confirms that the incident progressed beyond unauthorized access and privilege escalation to a full data breach involving successful exfiltration of sensitive server data.
