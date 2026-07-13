# SSH Authentication and Privilege Escalation Investigation

## Overview

This section documents the investigation of malicious SSH authentication activity, the compromise of the legitimate `developer` account, access to credential-related files, successful privilege escalation to root, and the creation of a privileged persistence account.

Analysis of the Linux `auth.log` identified malicious activity originating from `172.16.20.99`, the same IP address previously associated with web reconnaissance, the administrative portal credential attack, PHP web shell execution, and post-exploitation activity.

The authentication investigation demonstrates the attacker's progression from repeated SSH authentication attempts to successful access as `developer`, followed by abuse of the compromised account's existing sudo privileges to obtain full root access.

---

## 1. SSH Credential Attack

At approximately `08:50`, the attacker at `172.16.20.99` began a rapid sequence of SSH authentication attempts against multiple usernames.

Observed usernames included:

- `admin`
- `root`
- `test`
- `backup`
- `webmaster`
- `www-data`
- `ubuntu`
- `postgres`
- `mysql`
- `developer`
- `deploy`
- `git`
- `ftpuser`
- `service`

The authentication logs contained two important types of failure messages.

For nonexistent accounts:

`Invalid user admin`

For existing accounts:

`Failed password for root`

`Failed password for developer`

### Analyst Assessment

The activity is consistent with an automated SSH credential attack combined with username enumeration.

The distinction between `Invalid user` and `Failed password` can reveal whether a username exists on the target system. In this case, the logs indicate that accounts such as `root` and `developer` were valid targets.

**Confidence:** High

---

## 2. Repeated Authentication Attempts Against `developer`

At approximately `08:50:23`, the attacker began targeting the `developer` account with repeated password attempts.

Observed events included:

`Failed password for developer from 172.16.20.99`

Additional failures followed, including events indicating that the maximum number of authentication attempts had been exceeded.

The attacker continued attempting authentication against the account.

### Analyst Assessment

The repeated authentication failures demonstrate that `developer` was specifically targeted as part of the SSH credential attack.

However, the available logs do not prove that the correct password was ultimately discovered through brute force alone. Other possibilities include credentials obtained through previous post-exploitation activity, exposed configuration files, credential-related files, or another unknown source.

Therefore, the investigation distinguishes between the confirmed sequence of failed authentication attempts and the unknown exact source of the successful credentials.

**Confidence:** High for the credential attack; unconfirmed regarding the exact source of the successful password.

---

## 3. Successful Compromise of the `developer` Account

At `08:55:01`, the following event was recorded:

`Accepted password for developer from 172.16.20.99`

A session was subsequently opened for the `developer` account.

### Analyst Assessment

This event confirms successful SSH authentication using the legitimate `developer` account from the same IP address responsible for the earlier malicious activity.

The timeline demonstrates:

**Multiple failed SSH attempts → Successful password authentication → Interactive session as `developer`**

This represents confirmed account compromise.

**Confidence:** Confirmed

---

## 4. Access to `/etc/shadow` Using Sudo

Only seconds after the successful SSH login, at approximately `08:55:10`, the `developer` account executed:

`/bin/cat /etc/shadow`

The authentication logs recorded the command with:

`USER=root`

and confirmed that a sudo session was opened as UID `0`.

### Analyst Assessment

Unlike the earlier attempt to access `/etc/shadow` through the PHP web shell, the authentication logs directly confirm that this command was executed with root privileges through sudo.

The `/etc/shadow` file contains password hashes for local Linux accounts and is normally restricted to privileged access.

Access to this file by a compromised account represents significant credential access activity and could potentially enable offline password cracking or further account compromise.

**Confidence:** Confirmed

---

## 5. Root Privilege Escalation

At `08:56:02`, the compromised `developer` account executed:

`/bin/su -`

The logs subsequently recorded:

`Successful su for root by developer`

and:

`session opened for user root(uid=0) by developer(uid=1001)`

### Analyst Assessment

The attacker successfully obtained an interactive root shell.

The evidence indicates that the attacker abused the existing sudo privileges of the compromised `developer` account. The available logs do not demonstrate exploitation of a specific Linux vulnerability.

Therefore, the most accurate assessment is:

> The attacker abused the compromised `developer` account's existing sudo privileges to obtain an interactive root shell.

This distinction is important because the attacker achieved privilege escalation through abuse of valid account permissions rather than through a confirmed software exploit.

**Confidence:** Confirmed

---

## 6. Second Root Session

The first root session was closed at approximately `08:58:01`.

Four seconds later, at `08:58:05`, another successful root session was opened by `developer`.

The authentication logs again recorded:

`Successful su for root by developer`

### Analyst Assessment

The attacker obtained root access on at least two separate occasions during the compromised SSH session.

This reinforces the conclusion that the compromised `developer` account had sufficient sudo privileges to repeatedly obtain a root shell.

**Confidence:** Confirmed

---

## 7. Persistence Through `svc_monitor`

At `09:05:01`, a new local account named `svc_monitor` was created.

The authentication logs recorded:

- Username: `svc_monitor`
- UID: `1003`
- GID: `1003`
- Home directory: `/home/svc_monitor`
- Login shell: `/bin/bash`

At `09:05:08`, a password was assigned to the account.

At `09:05:15`, the account was added to the `sudo` group.

### Cross-Source Correlation

The Apache access logs recorded commands sent through the PHP web shell to:

1. Create the `svc_monitor` account.
2. Assign a password.
3. Add the account to the `sudo` group.

The Linux authentication logs independently confirmed that these system-level changes occurred successfully.

The original password value has been intentionally redacted from this public portfolio project.

### Analyst Assessment

The creation of a new interactive account with sudo privileges represents a persistence mechanism that could allow continued privileged access to the compromised server.

The account name `svc_monitor` resembles a legitimate service or monitoring account, potentially reducing the likelihood of immediate suspicion during a superficial account review.

**Confidence:** Confirmed

---

## 8. Timeline Anomaly Involving `svc_monitor`

A significant timestamp inconsistency was identified during correlation with the FTP logs.

The observed sequence was:

| Timestamp | Event |
|---|---|
| `09:02:16` | Successful FTP authentication as `svc_monitor` |
| `09:05:01` | `svc_monitor` recorded as newly created |
| `09:05:08` | Password assigned to `svc_monitor` |
| `09:05:15` | `svc_monitor` added to the `sudo` group |

### Analyst Assessment

According to the available timestamps, the account successfully authenticated to FTP approximately three minutes before its recorded creation.

The available evidence does not provide a definitive explanation for this discrepancy.

Possible explanations could include:

- Timestamp inconsistencies between log sources.
- Missing earlier account creation events.
- Limitations or inconsistencies in synthetic lab data.

None of these explanations can be confirmed from the available evidence.

This anomaly is therefore documented without forcing an unsupported conclusion.

---

## Attack Progression

The authentication investigation established the following progression:

**SSH Credential Attack → Targeting of `developer` → Successful SSH Authentication → Sudo Access to `/etc/shadow` → Interactive Root Shell → Second Root Session → Privileged Persistence Account Creation**

---

## Key Authentication Findings

- The attacker IP was identified as `172.16.20.99`.
- Multiple valid and invalid usernames were targeted over SSH.
- The legitimate `developer` account was successfully accessed using password authentication.
- The exact source of the successful `developer` credentials cannot be conclusively determined.
- The compromised account used sudo to access `/etc/shadow` as root.
- The attacker obtained an interactive root shell.
- A second root session was later established.
- The `svc_monitor` account was created and added to the `sudo` group for persistence.
- A timeline inconsistency involving `svc_monitor` was identified and documented.

---

## Splunk Investigation Evidence

The following evidence should be supported by actual Splunk screenshots and SPL queries:

1. SSH authentication attempts originating from `172.16.20.99`.
2. Enumeration of targeted usernames.
3. Failed password attempts against `developer`.
4. Successful SSH authentication as `developer`.
5. Sudo execution of `/bin/cat /etc/shadow`.
6. Successful privilege escalation to root.
7. The second root session.
8. Creation of the `svc_monitor` account.
9. Password assignment to `svc_monitor`.
10. Addition of `svc_monitor` to the `sudo` group.

> Screenshots and SPL queries should reflect the actual investigation performed in Splunk and should not be fabricated solely for presentation.

---

## Final Assessment

The Linux authentication logs confirm that the attacker at `172.16.20.99` conducted an SSH credential attack against multiple accounts and subsequently authenticated successfully as the legitimate `developer` user.

Within seconds of successful access, the compromised account used its existing sudo privileges to read `/etc/shadow` and obtain an interactive root shell. A second root session was later established.

The attacker also established privileged persistence by creating the `svc_monitor` account, assigning it a password, and adding it to the `sudo` group. These system changes were independently confirmed through correlation between Apache web shell activity and Linux authentication events.

The authentication evidence demonstrates a complete progression from account compromise to full administrative control of the Linux server.
