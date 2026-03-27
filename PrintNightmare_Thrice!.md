# PrintNightmare, Thrice! - TryHackMe Walkthrough

## Room Information
- **Room**: [PrintNightmare, thrice!](https://tryhackme.com/room/printnightmarec3kj)
- **Difficulty**: Medium
- **Category**: Incident Response / Forensics / Windows Print Spooler

## Scenario
After discovering the PrintNightmare attack, the security team pushed an emergency patch to all endpoints. The PrintNightmare exploit used previously no longer works. However, the same 2 employees discovered yet another exploit that can possibly work on a fully patched endpoint to elevate their privileges.

**Task**: Inspect the artifacts on the endpoint to detect the PrintNightmare exploit used.

## Tools Required
- Wireshark
- Brim
- FullEventLogView
- Process Monitor (ProcMon)

---

## Solution

### Question 1: What remote address did the employee navigate to?
**Answer**: `20.188.56.147`

**Solution**: Open the PCAP file in Wireshark, go to Statistics → Conversations → IPv4, and sort by packet count. The IP with the most communications is the answer.

---

### Question 2: Per the PCAP, which user returns a STATUS_LOGON_FAILURE error?
**Answer**: `THM-PRINTNIGHT0\rjones`

**Solution**: Filter in Wireshark using `smb2.nt_status == 0xc000006d` to find failed login attempts, then follow the TCP stream to find the username.

---

### Question 3: Which user successfully connects to an SMB share?
**Answer**: `THM-PRINTNIGHT0/gentilguest`

**Solution**: Filter using `smb2.nt_status == 0x00000000` to find successful connections. Check the Session Setup Response for the account name.

---

### Question 4: What is the first remote SMB share the endpoint connected to? What was the first filename? What was the second?
**Answer**: `\\printnightmare.gentilkiwi.com\IPC$,srvsvc,spoolss`

**Solution**: In Brim, use filter `_path=~smb* OR _path=dce_rpc | sort ts` to find the first SMB share and filenames.

---

### Question 5: From which remote SMB share was malicious DLL obtained? What was the path to the remote folder for the first DLL? How about the second?
**Answer**: `\\printnightmare.gentilkiwi.com\print$,x64\3\mimispool.dll,W32X86\3\mimispool.dll`

**Solution**: Use filter `_path=~smb* OR _path=dce_rpc AND name=*.dll | sort ts` to find DLL files. The mimispool.dll (sounds like mimikatz) is the malicious DLL.

---

### Question 6: What was the first location the malicious DLL was downloaded to on the endpoint? What was the second?
**Answer**: `C:\Windows\System32\spool\drivers\x64\3,C:\Windows\System32\spool\drivers\W32X86\3`

**Solution**: Use FullEventLogView with "Advanced Options" set to find events within 999 days, then search for mimispool.dll.

---

### Question 7: What is the folder that has the name of the remote printer server the user connected to?
**Answer**: `C:\Windows\System32\spool\SERVERS\printnightmare.gentilkiwi.com`

**Solution**: Explore the directory where malicious DLLs were found and look for the folder named after the remote printer server.

---

### Question 8: What is the name of the printer the DLL added?
**Answer**: `Kiwi Legit Printer`

**Solution**: While exploring the DLL download locations, identify the printer name added by the malicious DLL.

---

### Question 9: What was the process ID for the elevated command prompt? What was its parent process?
**Answer**: `5408,spoolsv.exe`

**Solution**: Use Process Monitor (ProcMon) to filter for cmd.exe processes. Find the elevated command prompt and note its PID and parent process.

---

### Question 10: What command did the user perform to elevate privileges?
**Answer**: `net localgroup administrators rjones /add`

**Solution**: Look up the command prompt PID (5408) in FullEventLogView to find the executed command.

---

## Summary of Findings
1. **Attacker IP**: 20.188.56.147
2. **Failed login user**: rjones
3. **Successful SMB user**: gentilguest
4. **Malicious DLL**: mimispool.dll (Mimikatz printer spooler DLL)
5. **Privilege escalation command**: Added user rjones to Administrators group
6. **Printer name**: Kiwi Legit Printer

---

## Technical Details
**CVE**: CVE-2021-34527 (PrintNightmare)

The PrintNightmare vulnerability allows low-privileged users to escalate to administrator by exploiting the Windows Print Spooler service, which runs as NT AUTHORITY\SYSTEM.

---

