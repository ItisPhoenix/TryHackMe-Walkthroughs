# Boiler CTF - Walkthrough

**Target Machine:** 10.49.158.145
**Difficulty:** Intermediate

## Reconnaissance

### 1. Port Scanning
Initial scan using Nmap reveals several open ports:
- **Port 21:** FTP (vsftpd 3.0.3) - Anonymous login allowed.
- **Port 80:** HTTP (Apache 2.4.18)
- Port 10000: Webmin (MiniServ 1.930)
- **Port 55007:** SSH (OpenSSH 7.2p2)

### 2. Service Enumeration

#### FTP (Port 21)
Anonymous login is enabled. Listing the files shows a hidden directory `/pub/` containing a hidden file `.info.txt`.
Content of `.info.txt` (ROT13 encoded):
`Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!`
Decoded: `Just wanted to see if you find it. Lol. Remember: Enumeration is the key!`

**Part 1 Answers:**
- **Q1:** .txt
- **Q2:** ssh

#### HTTP (Port 80)
The web server has a standard Apache index page. Checking `robots.txt` reveals several paths, but most are red herrings.
Decoding the numbers in `robots.txt` gives an MD5 hash `99b0660cd95adea327c54182baa51584`, which decodes to "kidding".

Further enumeration reveals a `/joomla/` directory.
- **Q5:** Joomla

Within `/joomla/`, a hidden directory `/_test/` is discovered running **sar2html version 3.2.1**.

## Exploitation

### 1. Remote Code Execution (RCE)
`sar2html` is vulnerable to OS Command Injection via the `plot` parameter.
Testing the vulnerability: `http://10.49.158.145/joomla/_test/index.php?plot=;ls`

Listing the directory reveals an interesting file:
- **Q7:** log.txt

Reading `log.txt`:
`basterd : superduperp@$$`

### 2. System Enumeration
Using the RCE to list users in `/etc/passwd`:
- Users found: `stoner`, `basterd`.

Searching for credentials in `basterd`'s home directory reveals `backup.sh`:
- **Part 2 Q1:** backup
Content of `backup.sh` reveals stoner's password: `superduperp@$$no1knows`

## Privilege Escalation

### 1. Escalation to Root
Searching for SUID binaries using the RCE:
`find / -perm -u=s -type f 2>/dev/null`
Binary found: `/usr/bin/find`
- **Part 2 Q3:** find

Exploiting the SUID `find`:
`/usr/bin/find /root/root.txt -exec cat {} \;`

## Flags

- **Part 2 Q2 (user.txt):** You made it till here, well done.
- **Part 2 Q4 (root.txt):** It wasn't that hard, was it?

---

### Questions Summary

**Part 1:**
1. File extension after anon login: **.txt**
2. File extension after anon login: **ssh**
3. What's running on port 10000? **Webmin**
4. Can you exploit the service running on that port? **yay**
5. What's CMS can you access? **Joomla**
6. Interesting file name in the folder? **log.txt**

**Part 2:**
1. Where was the other users pass stored? **backup**
2. user.txt: **You made it till here, well done.**
3. What did you exploit to get the privileged user? **find**
4. root.txt: **It wasn't that hard, was it?**
