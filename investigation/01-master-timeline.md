# Master Attack Timeline

## Overview

This timeline reconstructs the attack against the Linux server `brightstar-web` by correlating events across three independent log sources: Apache `access.log`, Linux `auth.log`, and `vsftpd.log`.

The primary malicious activity originated from `172.16.20.99` and progressed from reconnaissance and credential attacks to remote command execution, root access, persistence, and confirmed data exfiltration.

> **Analytical Note:** This timeline distinguishes between confirmed evidence and analytical assessments. Where the available logs do not directly prove an action or outcome, the confidence level is reduced accordingly.

---

## Complete Attack Timeline

| Timestamp | Attack Stage | Event | Evidence | Log Source | Confidence |
|---|---|---|---|---|---|
| 08:25:21 | FTP Reconnaissance | The attacker tested FTP access using the `admin` account. | `FAIL LOGIN` from `172.16.20.99` | `vsftpd.log` | High |
| 08:25:25 | FTP Reconnaissance | The attacker tested FTP access using the `root` account. | `FAIL LOGIN` from `172.16.20.99` | `vsftpd.log` | High |
| 08:25 | Active Reconnaissance | Nmap Scripting Engine activity was observed against the web server, probing resources such as `/robots.txt`, `/sitemap.xml`, and `/.git/HEAD`. | User-Agent identified as `Nmap Scripting Engine` | `access.log` | High |
| 08:30 | Vulnerability Scanning | Nikto scanned the web server and identified several accessible resources, including `/admin/login`, `/phpinfo.php`, `/phpmyadmin/`, and `/uploads/`. | User-Agent identified as `Nikto/2.5.0` | `access.log` | High |
| 08:35 | Directory Enumeration | Gobuster enumerated web directories and endpoints, including `/admin`, `/api`, `/uploads`, `/login`, and `/dashboard`. | User-Agent identified as `gobuster/3.6` | `access.log` | High |
| 08:40:01 | Credential Attack | Hydra began a rapid credential attack against `/admin/login`, generating repeated `401 Unauthorized` responses. | Multiple POST requests with Hydra User-Agent | `access.log` | High |
| 08:40:14 | Suspected Admin Compromise | A Hydra request to `/admin/login` returned HTTP `200` after repeated `401` responses. Subsequent access to administrative pages strongly indicates successful authentication. | Transition from HTTP `401` to `200`, followed by admin activity | `access.log` | High |
| 08:45 | Administrative Access | The attacker accessed administrative resources including `/admin/dashboard`, `/admin/settings`, `/admin/users`, and `/admin/upload`. | Successful HTTP requests from `172.16.20.99` | `access.log` | High |
| 08:45 | Web Shell Deployment | A file upload was followed by requests to `/uploads/shell.php`, strongly indicating successful deployment of a PHP web shell. | `POST /admin/upload` followed by access to `/uploads/shell.php` | `access.log` | High |
| 08:47 | Remote Command Execution | The attacker executed commands through `shell.php`, including `id`, `whoami`, and `uname -a`. | Commands passed through the `cmd` URL parameter | `access.log` | High |
| 08:47–08:48 | System Discovery | The attacker performed user, process, network, and system enumeration using commands including `cat /etc/passwd`, `netstat -tlnp`, `ps aux`, and `ifconfig`. | Web shell command activity | `access.log` | High |
| 08:47 | Attempted Credential Access | The attacker attempted to read `/etc/shadow` through the web shell. The HTTP request returned `200`, but the available access log alone does not prove that the file contents were successfully retrieved. | `cat /etc/shadow` via `shell.php` | `access.log` | Medium |
| 08:48:18 | Sensitive File Discovery | The attacker searched the filesystem for SQL database files. | `find / -name *.sql -type f` | `access.log` | High |
| 08:48 | Sensitive File Discovery | The attacker searched the filesystem for PEM key files. | `find / -name *.pem -type f` | `access.log` | High |
| 08:48:32 | Sensitive Data Access | The attacker requested `/var/backups/customer_db.sql` through the web shell. The response size was `45,102` bytes, matching the size of the file later exfiltrated through FTP. | `cat /var/backups/customer_db.sql` → HTTP response size `45102` | `access.log` | High |
| 08:50:01 | Tool Transfer | The attacker downloaded `linpeas.sh` from `172.16.20.99:8080` to `/tmp/linpeas.sh`. | `wget` command executed through web shell | `access.log` | High |
| 08:50:08 | Tool Preparation | Execute permissions were added to the downloaded LinPEAS script. | `chmod +x /tmp/linpeas.sh` | `access.log` | High |
| 08:50:15 | Enumeration Tool Execution | The attacker executed LinPEAS, likely to identify privilege escalation opportunities. | `bash /tmp/linpeas.sh` | `access.log` | High |
| 08:50 | SSH Credential Attack | The attacker launched multiple SSH authentication attempts against valid and invalid usernames, including `root` and `developer`. | Repeated failed SSH authentication events from `172.16.20.99` | `auth.log` | High |
| 08:55:01 | Account Compromise | The attacker successfully authenticated over SSH as `developer` using a password. | `Accepted password for developer from 172.16.20.99` | `auth.log` | Confirmed |
| 08:55:10 | Credential Access | The compromised `developer` account used sudo to read `/etc/shadow` as root. | `COMMAND=/bin/cat /etc/shadow` with `USER=root` | `auth.log` | Confirmed |
| 08:56:02 | Root Privilege Escalation | The attacker used the compromised account's sudo privileges to obtain an interactive root shell. | `COMMAND=/bin/su -` followed by `Successful su for root by developer` | `auth.log` | Confirmed |
| 08:58:05 | Root Session | A second successful root session was opened by `developer`. | Second `Successful su for root by developer` event | `auth.log` | Confirmed |
| 09:00:02 | FTP Root Access | The attacker successfully authenticated to FTP as `root`. | `[root] OK LOGIN` from `172.16.20.99` | `vsftpd.log` | Confirmed |
| 09:00:05 | Data Exfiltration | The attacker successfully downloaded the customer database backup. | `OK DOWNLOAD: /var/backups/customer_db.sql`, `45102 bytes` | `vsftpd.log` | Confirmed |
| 09:00:18 | Data Exfiltration | The attacker successfully downloaded a potentially sensitive private key file. | `OK DOWNLOAD: /etc/ssl/private/payment_keys.pem`, `3247 bytes` | `vsftpd.log` | Confirmed |
| 09:00:25 | Data Exfiltration | The attacker successfully downloaded employee-related data. | `OK DOWNLOAD: /var/backups/employee_data.csv`, `18934 bytes` | `vsftpd.log` | Confirmed |
| 09:00:32 | Data Exfiltration | The attacker successfully downloaded the web application's configuration file. | `OK DOWNLOAD: /var/www/html/config.php`, `423 bytes` | `vsftpd.log` | Confirmed |
| 09:00:40 | Suspicious File Upload | The attacker uploaded `beacon.sh` to the hidden directory `/tmp/.hidden/`. Its name and location suggest possible beaconing, persistence, or C2 functionality, but its contents and execution are not available in the provided evidence. | `OK UPLOAD: /tmp/.hidden/beacon.sh` | `vsftpd.log` | High |
| 09:02:16 | Persistence Account Activity | Successful FTP authentication was observed using the `svc_monitor` account. This event conflicts with the recorded account creation time of `09:05:01`. | `[svc_monitor] OK LOGIN` | `vsftpd.log` | Confirmed event; timeline anomaly |
| 09:02:20 | Log File Download | The `svc_monitor` account downloaded `/var/log/auth.log`. | `OK DOWNLOAD` | `vsftpd.log` | Confirmed |
| 09:02:28 | Log File Download | The `svc_monitor` account downloaded `/var/log/apache2/access.log`. | `OK DOWNLOAD` | `vsftpd.log` | Confirmed |
| 09:05:01 | Persistence | A new local account named `svc_monitor` was created with `/bin/bash` as its shell. | `new user: name=svc_monitor` | `auth.log` | Confirmed |
| 09:05:08 | Persistence | A password was assigned to the `svc_monitor` account. | `password changed for svc_monitor` | `auth.log` | Confirmed |
| 09:05:15 | Privileged Persistence | The `svc_monitor` account was added to the `sudo` group. | `add 'svc_monitor' to group 'sudo'` | `auth.log` | Confirmed |

---

## Key Correlations

### 1. Database Discovery to Exfiltration

The attacker first searched for SQL files and accessed `/var/backups/customer_db.sql` through the deployed web shell. The HTTP response size was `45,102` bytes. Later, the same file was successfully downloaded through FTP with an identical size of `45,102` bytes.

This correlation demonstrates the progression:

**Discovery → Access → Exfiltration**

---

### 2. Web Shell Commands Confirmed by Authentication Logs

The Apache access logs recorded commands sent through the web shell to create the `svc_monitor` account, assign a password, and add it to the `sudo` group.

The Linux authentication logs independently confirmed that these account modifications occurred successfully.

This provides strong multi-source evidence of attacker-established persistence.

---

### 3. Web Compromise to SSH and Root Access

The attack progressed from web-based remote command execution to successful SSH access as `developer`, followed by abuse of the account's existing sudo privileges to obtain a root shell.

The exact method by which the attacker obtained the successful `developer` credentials cannot be conclusively determined from the available logs.

---

## Timeline Anomaly

A significant timestamp inconsistency was identified involving the `svc_monitor` account:

- `09:02:16` — Successful FTP authentication as `svc_monitor`.
- `09:05:01` — The `svc_monitor` account is recorded as newly created.
- `09:05:08` — A password is assigned to the account.
- `09:05:15` — The account is added to the `sudo` group.

The available evidence does not provide a definitive explanation for this discrepancy. Possible causes could include timestamp inconsistencies between log sources, synthetic lab data limitations, or missing prior events; however, none of these explanations can be confirmed from the available evidence.

---

## Final Timeline Assessment

The evidence demonstrates a multi-stage compromise that progressed from active reconnaissance to full root access and confirmed sensitive data exfiltration.

The attacker successfully:

1. Reconnoitered multiple exposed services.
2. Identified administrative web resources.
3. Conducted a credential attack against the administrative login.
4. Deployed a PHP web shell and achieved remote command execution.
5. Performed extensive system and sensitive-file discovery.
6. Transferred and executed LinPEAS.
7. Conducted SSH credential attacks and successfully accessed the `developer` account.
8. Abused sudo privileges to obtain root access.
9. Exfiltrated multiple sensitive files over FTP.
10. Uploaded a suspicious script into a hidden directory.
11. Established privileged persistence through the `svc_monitor` account.

The compromise should be classified as a **Critical Security Incident** due to confirmed root-level access, persistence, credential exposure, and successful sensitive data exfiltration.
