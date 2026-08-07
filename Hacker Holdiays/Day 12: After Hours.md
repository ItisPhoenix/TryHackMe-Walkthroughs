## Walkthrough: TryHackMe Room: "Room 404"

## Executive Summary

**Room Name:** After Hours  
**Category:** Forensics  
**Difficulty:** Medium

This challenge demonstrates a **fileless persistence** technique using the **Windows Management Instrumentation (WMI) Repository**.

Unlike traditional persistence mechanisms such as:

- Startup folders
- Registry Run keys
- Scheduled Tasks
- Windows Services

the attacker stores the malicious payload directly inside the **WMI Repository** as a **custom WMI class**. A PowerShell launcher retrieves the stored payload, decompresses it entirely in memory, and executes a .NET assembly without ever touching disk.

The investigation consists of three major stages:

1. Recover the hidden PowerShell launcher.
2. Extract the embedded .NET assembly from the custom WMI class.
3. Reverse engineer the assembly to recover the flag.

---

# Attack Overview

```
OBJECTS.DATA
      │
      ▼
Hidden PowerShell Launcher
      │
      ▼
ROOT\cimv2:Win32_HardwareTelemetry
      │
      ▼
Property: ConfigData
      │
      ▼
Base64
      │
      ▼
Raw Deflate
      │
      ▼
.NET Assembly
      │
      ▼
ILSpy
      │
      ▼
Hidden Base64 Flag
```

---

# Initial Enumeration

The room provides the following files:

```
challenge_attachments/
│── INDEX.BTR
│── MAPPING1.MAP
│── MAPPING2.MAP
│── MAPPING3.MAP
└── OBJECTS.DATA
```

These files together represent a Windows **WMI Repository**.

The most interesting file is:

```
OBJECTS.DATA
```

This contains:

- WMI class definitions
- WMI instances
- WMI properties
- Stored data

Everything else primarily supports indexing and mapping.

---

# Step 1 – Extract Readable Strings

Since the repository is binary, begin by extracting readable strings.

```bash
cd challenge_attachments

strings OBJECTS.DATA > objects.txt
```

Instead of looking for generic persistence indicators, search for PowerShell.

```bash
grep -i powershell objects.txt
```

---

## Finding the PowerShell Launcher

Among the results is a suspicious command:

```text
cmd /C powershell.exe -Sta -Nop -Window Hidden -enc
```

Immediately after `-enc` is a very long Base64 string.

This indicates that the attacker used PowerShell's **EncodedCommand** feature.

---

## Understanding the PowerShell

Decoding the Base64 reveals a PowerShell script that performs the following actions:

```powershell
$file = ([WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry')
            .Properties['ConfigData'].Value
```

This line immediately reveals the attacker is using a **custom WMI class**.

### Malicious Class

```
ROOT\cimv2
    ↓
Win32_HardwareTelemetry
```

### Property

```
ConfigData
```

The remainder of the script performs:

```powershell
Convert.FromBase64String()

↓

DeflateStream()

↓

Reflection.Assembly.Load()
```

This means:

```
ConfigData
        ↓
Base64
        ↓
Raw Deflate
        ↓
.NET Assembly
```

The payload never touches disk.

---

# Step 2 – Locate the Stored Payload

Search the extracted strings for the custom class.

```bash
grep -A5 -B5 "Win32_HardwareTelemetry" objects.txt
```

The output contains:

```
Win32_HardwareTelemetry

ConfigData

7VZPbFRF...
```

The long Base64 string beginning with:

```
7VZPbFRF...
```

is the payload stored inside the WMI repository.

Extract only that Base64 string.

---

# Step 3 – Recover the PE File

Open the Base64 in CyberChef.

Use the following recipe:

```
From Base64

↓

Raw Inflate
```

Do **NOT** use:

- GZip Inflate
- Zlib Inflate

The PowerShell specifically uses **DeflateStream**, so the correct operation is **Raw Inflate**.

After decompression the output begins with:

```
MZ
```

followed by

```
This program cannot be run in DOS mode.
```

This confirms the recovered payload is a Windows Portable Executable.

Save it as:

```
payload.exe
```

Verify:

```bash
file payload.exe
```

Output:

```
PE32 executable (GUI)
Intel 80386
Mono/.NET assembly
```

---

# Step 4 – Reverse Engineer the Assembly

Launch ILSpy.

```
./ILSpy
```

Open:

```
payload.exe
```

Expand:

```
payload

↓

AfterHours

↓

Program

↓

Main()
```

The Main method contains:

```csharp
if (Environment.MachineName == "bytelotusdc")
{
    ProcessStartInfo.Arguments =
        "/c net user patch VEhNe1A0dG.............QmFjS2QwMH0= /add";
}
```

Notice the suspicious string:

```
VEhNe1A0dGN...........FjS2QwMH0=
```

This is another Base64 value.

---

# Step 5 – Decode the Final Payload

Decode the string:

```bash
echo 'VEhNe.................NfQmFjS2QwMH0=' | base64 -d
```

Result:

```
THM{...............}
```


---

# Attack Analysis

The attacker intentionally avoided common persistence locations.

Instead they abused the WMI Repository by creating a custom class:

```
ROOT\cimv2

↓

Win32_HardwareTelemetry
```

The PowerShell launcher simply retrieves the Base64 payload from the class property:

```
ConfigData
```

The payload is then:

```
Base64

↓

Raw Deflate

↓

Reflection.Assembly.Load()
```

Because the assembly is loaded directly into memory:

- No executable is written to disk.
- Autoruns does not detect it.
- Startup folders remain clean.
- Registry Run keys remain clean.
- Scheduled Tasks remain clean.

Only by examining the **raw WMI repository** can the persistence be discovered.

---

# MITRE ATT&CK Mapping

| Technique | Description |
|-----------|-------------|
| T1546.003 | WMI Event Subscription |
| T1059.001 | PowerShell |
| T1027 | Obfuscated Files or Information |
| T1620 | Reflective Code Loading |
| T1140 | Deobfuscate/Decode Files or Information |

---

# Key Takeaways

- WMI Repository is frequently overlooked during forensic investigations.
- PowerShell EncodedCommand is commonly used to hide malicious launchers.
- Base64 combined with Raw Deflate is a common technique for storing PE files.
- Reflection.Assembly.Load enables fileless execution of .NET assemblies.
- Reverse engineering recovered payloads is often necessary to obtain the final stage of an attack.
