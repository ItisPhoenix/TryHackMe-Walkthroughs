# TryHackMe: Lookup Walkthrough

## Machine Information
- **Target IP:** 10.48.138.197
- **Hostname:** `lookup.thm`, `files.lookup.thm`
- **Difficulty:** Easy
- **Goal:** Capture `user.txt` and `root.txt` flags.

---

## 1. Reconnaissance

### Nmap Scan
Initial scanning revealed two open ports:
- **22/tcp (SSH):** OpenSSH 8.2p1 Ubuntu
- **80/tcp (HTTP):** Apache httpd 2.4.41

### Web Enumeration
Accessing the web server at `http://10.48.138.197` redirects to `http://lookup.thm`.
A login page is presented.

### Virtual Host Discovery
Subdomain enumeration revealed `files.lookup.thm`. Both `lookup.thm` and `files.lookup.thm` were added to `/etc/hosts`.

---

## 2. Initial Access

### Login Brute-force
The username `jose` was found to be valid. A password brute-force attack successfully identified the credentials:
- **Credentials:** `jose:password123`

### Web Application Exploitation
After logging in, I was redirected to `http://files.lookup.thm/elFinder/elfinder.html`.
The application is running **elFinder v2.1.47**, which is vulnerable to **Command Injection (CVE-2019-9194)** via the `resize` (rotate) command.

### Exploitation Steps
1.  **Upload a malicious file:** Uploaded a valid JPEG file renamed to `test.jpg;sleep 10;test.jpg`.
2.  **Trigger Injection:** Executed the `resize` command with `mode=rotate` and `degree=180` on the uploaded file.
3.  **Confirmation:** The server response time was delayed by 10 seconds, confirming RCE.

---

## 3. Privilege Escalation (User)

### Finding User Password
By exploring the system via command injection, I discovered a file named `credentials.txt` in the web directory containing:
`think : nopassword`

### SSH Access
I logged in via SSH as the user `think`.

### User Flag
The user flag was found in the home directory:
- **User Flag:** `38375fb4dd8baa2b2039ac03d92b820e`

---

## 4. Privilege Escalation (Root)

### Local Enumeration
Checking sudo privileges with `sudo -l` revealed that `think` can run `/usr/bin/look` as root without a password.

### Exploiting `look`
The `look` command can be used to read files. I leveraged this to read the root flag:
```bash
sudo /usr/bin/look "" /root/root.txt
```

### Root Flag
- **Root Flag:** `5a285a9f257e45c68bb6c9f9f57d18e8`
