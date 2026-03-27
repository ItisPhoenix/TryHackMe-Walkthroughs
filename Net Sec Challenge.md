# Net Sec Challenge Walkthrough - TryHackMe

**Target IP:** 10.49.138.10

## Initial Enumeration
The first step was a full port scan using Nmap to identify open services and any non-standard ports.
```bash
nmap -p- -sV -sC -T4 10.49.138.10
```

### Scan Results:
- **Port 22:** SSH (OpenSSH 8.2p1) - **Flag Found in Banner:** `THM{946219583339}`
- **Port 80:** HTTP (lighttpd) - **Flag Found in Header:** `THM{web_server_25352}`
- **Port 139/445:** Samba (smbd 4)
- **Port 8080:** HTTP (Node.js/Express) - IDS Challenge
- **Port 10021:** FTP (vsftpd 3.0.5)

---

## Solutions

### Question 1: Highest port number open less than 10,000?
**Answer:** `8080`
*From the Nmap scan, 8080 is the highest port under 10,000.*

### Question 2: Open port outside common 1000 above 10,000?
**Answer:** `10021`
*The Nmap scan revealed port 10021.*

### Question 3: How many TCP ports are open?
**Answer:** `6`
*Ports: 22, 80, 139, 445, 8080, 10021.*

### Question 4: Flag hidden in HTTP server header?
**Answer:** `THM{web_server_25352}`
*Retrieved via `curl -I 10.49.138.10` or the Nmap service scan.*

### Question 5: Flag hidden in SSH server header?
**Answer:** `THM{946219583339}`
*Retrieved by connecting to SSH via `nc 10.49.138.10 22` or observing the Nmap version scan output.*

### Question 6: Version of FTP server listening on nonstandard port?
**Answer:** `vsftpd 3.0.5`
*Identified on port 10021 during service enumeration.*

### Question 7: Flag hidden in account files via FTP?
**Answer:** `THM{321452667098}`
- **Discovery:** Usernames `eddie` and `quinn` were identified.
- **Attack:** Brute-force using `hydra` revealed credentials `quinn:andrea`.
- **Exploitation:** Logged in to FTP on port 10021 and retrieved `ftp_flag.txt`.
```bash
curl -s ftp://10.49.138.10:10021/ftp_flag.txt -u quinn:andrea
```

### Question 8: Flag from http://10.49.138.10:8080 challenge?
**Answer:** `THM{f7443f99}`
- **Challenge:** The web app detects Nmap scans and increments a counter.
- **Solution:** Performed a stealthy scan (Null, FIN, or Xmas) to bypass the IDS monitoring.
```bash
sudo nmap -sN 10.49.138.10
```
*After the scan completed without detection, the flag appeared on the challenge page.*

---

