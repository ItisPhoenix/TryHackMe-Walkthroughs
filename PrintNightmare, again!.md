# PrintNightmare, again! - TryHackMe Walkthrough

## Room Information
- **Room**: [PrintNightmare, again!](https://tryhackme.com/room/printnightmarec2bn7l)
- **Difficulty**: Easy
- **Category**: Incident Response / Forensics / Windows Print Spooler
- **CVE**: CVE-2021-1675 (PrintNightmare)

## Scenario
In the weekly internal security meeting, it was reported that an employee overheard two co-workers discussing the PrintNightmare exploit and how they can use it to elevate their privileges on their local computers.

**Task**: Inspect the artifacts on the endpoint to detect the exploit they used.

## Tools Required
- FullEventLogView
- Process Monitor (ProcMon)
- ProcDOT (optional)

---

## Solution

### Question 1: The user downloaded a zip file. What was the zip file saved as?
**Answer**: `levelup.zip`

**Solution**: Open FullEventLogView, go to Options > Advanced Options and set "Show events from all times". Search for ".zip" in the events. The file was downloaded to `C:\Users\bmurphy\Downloads\levelup.zip`.

---

### Question 2: What is the full path to the exploit the user executed?
**Answer**: `C:\Users\bmurphy\Downloads\CVE-2021-1675-main\CVE-2021-1675.ps1`

**Solution**: Continue reviewing the FullEventLogView timeline to find the PowerShell script that was extracted and executed.

---

### Question 3: What was the temp location the malicious DLL was saved to?
**Answer**: `C:\Users\bmurphy\AppData\Local\Temp\3\nightmare.dll`

**Solution**: Look for DLL file creation events in the timeline. The malicious DLL was first downloaded to the user's Temp directory.

---

### Question 4: What was the full location the DLL loads from?
**Answer**: `C:\Windows\System32\spool\drivers\x64\3\New\nightmare.dll`

**Solution**: The Print Spooler service (spoolsv.exe) moved the malicious DLL to the printer drivers directory where it gets loaded.

---

### Question 5: What is the primary registry path associated with this attack?
**Answer**: `HKLM\System\CurrentControlSet\Control\Print\Environments\Windows x64\Drivers\Version-3\THMPrinter\DriverVersion`

**Solution**: Search for registry events in FullEventLogView (Event ID 13). This registry path shows where the malicious printer driver was registered.

---

### Question 6: What was the PID for the process that would have been blocked from loading a non-Microsoft-signed binary?
**Answer**: `2600`

**Solution**: Look for Microsoft-Windows-Security-Mitigations events. The process `spoolsv.exe` (PID 2600) would have been blocked from loading the non-Microsoft-signed DLL.

---

### Question 7: What is the username of the newly created local administrator account?
**Answer**: `backup`

**Solution**: Search for Event ID 4720 (user account created) in FullEventLogView, or examine the PowerShell history file.

---

### Question 8: What is the password for this user?
**Answer**: `ucGGDMyFHkqMRWwHtQ`

**Solution**: Access the PowerShell console history at:
```
C:\Users\bmurphy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
The password is visible in the Invoke-Nightmare command.

---

### Question 9: What two commands did the user execute to cover their tracks?
**Answer**: `rmdir .\CVE-2021-1675-main\,del .\levelup.zip`

**Solution**: Review the PowerShell history file. After completing the privilege escalation, the attacker deleted the exploit files to cover their tracks.

---

## Attack Timeline

| Time | Event |
|------|-------|
| 09:52:07 | User downloads `levelup.zip` to Downloads folder |
| 09:52:27 | User extracts zip and accesses exploit directory |
| 09:53:38 | Malicious DLL saved to Temp, then moved to spool directory |
| 09:53:38 | Registry keys created for fake printer driver |
| 09:53:38 | New local administrator account created |

## Technical Details

### CVE-2021-1675 (PrintNightmare)
CVE-2021-1675 is a local privilege escalation vulnerability in the Windows Print Spooler service. When exploited, it allows a low-privileged user to execute arbitrary code with SYSTEM privileges.

The vulnerability was originally thought to require local access, but researchers discovered it could be exploited remotely as well, leading to CVE-2021-34527.

### Attack Vector
1. Attacker downloads the PrintNightmare exploit (CVE-2021-1675)
2. Executes PowerShell script to leverage the Print Spooler vulnerability
3. Malicious DLL is loaded by spoolsv.exe (running as SYSTEM)
4. New local administrator account created
5. Attacker cleans up evidence

### Indicators of Compromise (IOCs)
- Malicious DLL: `nightmare.dll`, `Newnightmare.dll`
- Fake printer: `THMPrinter`
- New user account: `backup`
- Registry modification: `HKLM\System\CurrentControlSet\Control\Print\...`

---

## Mitigation Strategies
1. Disable Windows Print Spooler service on systems that don't require printing
2. Apply Microsoft security patches for CVE-2021-1675 and CVE-2021-34527
3. Implement endpoint detection and response (EDR) solutions
4. Monitor for suspicious printer driver installations
5. Review Windows Event Logs for Event ID 316, 808, 811, 31017, 7031

---

