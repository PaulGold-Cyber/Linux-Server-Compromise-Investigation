# Splunk Investigation and SPL Queries

## Overview

This investigation was conducted in Splunk using the following dataset:

```spl
index=lab08 source=srv8
```

The available dataset contained a relatively small number of events across three primary log sources:

- Apache `access.log`
- Linux `auth.log`
- FTP `vsftpd.log`

Because the dataset was relatively small, the investigation began with a complete chronological review of the available events rather than relying exclusively on complex SPL queries.

This approach allowed the attack sequence to be reconstructed from beginning to end. After identifying suspicious activity, targeted searches were used to isolate specific indicators, accounts, files, tools, and attack behaviors.

The investigation focused on three main activities:

1. Chronological log review.
2. Targeted keyword and indicator searches.
3. Correlation of related events across multiple log sources.

---

## 1. Initial Dataset Review

The investigation began by reviewing all available events:

```spl
index=lab08 source=srv8
| sort 0 _time
```

**Purpose:** Review the complete dataset chronologically and identify unusual patterns, suspicious IP addresses, authentication activity, offensive tools, file transfers, and command execution.

**Finding:** This initial review revealed a multi-stage attack originating from `172.16.20.99`.

The observed activity included:

- Nmap reconnaissance
- Nikto vulnerability scanning
- Gobuster directory enumeration
- Hydra credential attacks
- Administrative web access
- PHP web shell activity
- SSH authentication attacks
- Successful compromise of the `developer` account
- Root privilege escalation
- FTP-based data exfiltration
- Creation of the `svc_monitor` persistence account

---

## 2. Search for All Attacker IP Activity

```spl
index=lab08 source=srv8 "172.16.20.99"
| sort 0 _time
```

**Purpose:** Identify and correlate all activity associated with the primary attacker IP.

**Finding:** The same IP address appeared across web, authentication, and FTP activity, allowing the attack to be correlated across multiple log sources.

---

## 3. Review Apache Web Activity

```spl
index=lab08 host=srv8
| sort 0 _time
```

**Purpose:** Review web requests chronologically and identify reconnaissance, vulnerability scanning, credential attacks, administrative access, web shell activity, and post-exploitation commands.

> **Note:** If the Apache events in the Splunk environment use a different sourcetype name, the exact value should be replaced with the value configured in the lab environment.

---

## 4. Identify Offensive Security Tools

```spl
index=lab08 source=srv8
("Nmap Scripting Engine" OR "Nikto" OR "gobuster" OR "Hydra" OR "linpeas")
| sort 0 _time
```

**Purpose:** Identify events associated with offensive security and enumeration tools observed during the attack.

**Finding:** The attacker used multiple tools throughout different phases of the compromise:

- Nmap for active reconnaissance.
- Nikto for web vulnerability scanning.
- Gobuster for directory enumeration.
- Hydra for credential attacks.
- LinPEAS for Linux privilege escalation enumeration.

---

## 5. Identify Web Shell Activity

```spl
index=lab08 source=srv8 "shell.php"
| sort 0 _time
```

**Purpose:** Identify requests involving the malicious PHP web shell.

**Finding:** The attacker interacted with `/uploads/shell.php` and supplied Linux commands through the `cmd` parameter.

A more targeted search can also be used:

```spl
index=lab08 source=srv8 "shell.php" "cmd="
| sort 0 _time
```

---

## 6. Identify Sensitive File Discovery and Access

```spl
index=lab08 source=srv8
("customer_db.sql" OR "payment_keys.pem" OR "employee_data.csv" OR "config.php" OR "/etc/shadow")
| sort 0 _time
```

**Purpose:** Identify activity involving sensitive files accessed, searched for, or exfiltrated during the compromise.

**Finding:** The attacker targeted database backups, potential private key material, employee-related data, application configuration, and Linux password hashes.

---

## 7. Investigate SSH Activity Involving `developer`

```spl
index=lab08 source=srv8 "developer"
| sort 0 _time
```

**Purpose:** Reconstruct all events involving the compromised `developer` account.

**Finding:** The account was targeted by repeated SSH authentication attempts, successfully accessed using password authentication, and subsequently used to execute privileged commands and obtain a root shell.

---

## 8. Identify Failed and Successful SSH Authentication

```spl
index=lab08 source=srv8
("Failed password" OR "Accepted password")
| sort 0 _time
```

**Purpose:** Identify SSH authentication failures and successful logins.

**Finding:** Multiple failed authentication attempts were followed by successful authentication as `developer` from the attacker IP.

A more focused version can be used:

```spl
index=lab08 source=srv8 "172.16.20.99"
("Failed password" OR "Accepted password")
| sort 0 _time
```

---

## 9. Identify Root Privilege Escalation

```spl
index=lab08 source=srv8
("Successful su for root" OR "COMMAND=/bin/su -" OR "USER=root")
| sort 0 _time
```

**Purpose:** Identify commands and authentication events associated with privilege escalation to root.

**Finding:** The compromised `developer` account used existing sudo privileges to access `/etc/shadow` and successfully obtain an interactive root shell.

---

## 10. Investigate `svc_monitor` Persistence

```spl
index=lab08 source=srv8 "svc_monitor"
| sort 0 _time
```

**Purpose:** Identify all activity involving the attacker-created persistence account.

**Finding:** The search identified:

- Account creation.
- Password assignment.
- Addition to the `sudo` group.
- Successful FTP authentication.
- Downloads of authentication and Apache access logs.

The chronological results also revealed a timestamp anomaly: FTP authentication occurred before the account's recorded creation time.

---

## 11. Identify FTP Downloads and Uploads

```spl
index=lab08 source=srv8
("OK DOWNLOAD" OR "OK UPLOAD" OR "OK LOGIN")
| sort 0 _time
```

**Purpose:** Identify successful FTP authentication, file downloads, and uploads.

**Finding:** The attacker authenticated as `root`, successfully exfiltrated multiple sensitive files, and uploaded the suspicious `/tmp/.hidden/beacon.sh` script.

---

## 12. Search for Confirmed Exfiltration

```spl
index=lab08 source=srv8
("customer_db.sql" OR "payment_keys.pem" OR "employee_data.csv" OR "config.php")
"OK DOWNLOAD"
| sort 0 _time
```

**Purpose:** Isolate confirmed FTP downloads involving sensitive files.

**Finding:** Four sensitive files were successfully downloaded from the compromised server.

---

## 13. Correlate the Customer Database Across Log Sources

```spl
index=lab08 source=srv8 "customer_db.sql"
| sort 0 _time
```

**Purpose:** Trace the complete lifecycle of the customer database backup across the available telemetry.

**Finding:** The attacker:

1. Searched for SQL files.
2. Accessed `/var/backups/customer_db.sql` through the web shell.
3. Later successfully downloaded the same file through FTP.

The identical size of `45,102` bytes across the Apache and FTP evidence provided strong cross-source correlation.

---

## Investigation Methodology

The investigation did not depend on highly complex SPL because the dataset was small enough to review directly.

The primary methodology was:

**Full chronological review → Identify suspicious activity → Isolate the attacker IP → Search for specific accounts, files, and tools → Correlate events across log sources → Reconstruct the complete attack timeline**

This approach was appropriate for the available dataset and helped identify several important correlations, including:

- Web reconnaissance progressing to administrative compromise.
- Web shell commands followed by confirmed operating system changes.
- Failed SSH authentication followed by successful access as `developer`.
- Root privilege escalation shortly after account compromise.
- Discovery and access of `customer_db.sql` followed by confirmed FTP exfiltration.
- Creation of `svc_monitor` confirmed across multiple log sources.
- A timestamp inconsistency involving `svc_monitor` identified through chronological correlation.

---

## Detection Queries Developed After the Investigation

The following queries were developed based on behaviors identified during the investigation. They are intended as detection examples rather than queries necessarily used during the original analysis.

### Multiple SSH Failures Followed by Success

```spl
index=lab08 source=srv8
("Failed password" OR "Accepted password")
| rex field=_raw "(?<auth_result>Failed password|Accepted password)"
| stats count(eval(auth_result="Failed password")) AS failures
        count(eval(auth_result="Accepted password")) AS successes
        by src_ip, user
| where failures >= 5 AND successes >= 1
```

**Detection Goal:** Identify accounts that experience multiple failed SSH authentication attempts followed by at least one successful login.

> This query may require field extraction adjustments depending on how `src_ip` and `user` are parsed in the Splunk environment.

---

### New Account Added to the Sudo Group

```spl
index=lab08 source=srv8
("new user:" OR "add '" AND "to group 'sudo'")
| sort 0 _time
```

**Detection Goal:** Identify creation of new local accounts and additions to the privileged `sudo` group.

---

### Suspicious PHP Execution from Upload Directory

```spl
index=lab08 source=srv8 "/uploads/" ".php"
| sort 0 _time
```

**Detection Goal:** Identify requests to PHP files located inside upload directories, which may indicate web shell deployment or execution.

---

### Sensitive File Downloads Through FTP

```spl
index=lab08 source=srv8 "OK DOWNLOAD"
(".sql" OR ".pem" OR ".csv" OR "config.php" OR "auth.log" OR "access.log")
| sort 0 _time
```

**Detection Goal:** Identify successful FTP downloads involving potentially sensitive file types or known high-value files.

---

## Final Note

The investigation demonstrates that effective SOC analysis does not always require complex queries.

For a relatively small dataset, a complete chronological review combined with focused searches and cross-source correlation can be more effective than unnecessarily complex SPL.

The most important analytical outcome was not the complexity of individual queries, but the ability to connect events across Apache, Linux authentication, and FTP logs to reconstruct the full attack chain from reconnaissance to root compromise, persistence, and confirmed data exfiltration.
