### Overview
This task involves analyzing a multi-stage malware delivery chain that begins with a Valentine's Day e-card and culminates in data exfiltration. The attack uses multiple domains all pointing to 10.48.135.117, with payloads designed to only execute on Windows systems to evade automated analysis.

### Step-by-Step Solution

#### 1. Initial Setup
- Extract the provided `loveletter.zip` using password `happyvalentines`
- The zip contains an email message that reveals the attack infrastructure

#### 2. Domain Configuration
All attacker-controlled domains point to 10.48.135.117. Added these to `/etc/hosts`:
```
10.48.135.117 delivery.cupidsarrow.thm
10.48.135.117 gifts.bemyvalentine.thm
10.48.135.117 api.valentinesforever.thm
10.48.135.117 loader.sweethearts.thm
10.48.135.117 cdn.loveletters.thm
10.48.135.117 lovetters.thm
```

#### 3. Stage 1: JavaScript Decoy
- Accessed `http://delivery.cupidsarrow.thm/valentine-animations.js` with Windows User-Agent
- Downloaded large Base64-encoded JavaScript payload
- Decoded using Python: Base64 decode → XOR with key "VALENTINE"
- Resulted in `LOVE_LETTER.pdf.iso` (ISO 9660 filesystem)

#### 4. Stage 2: ISO Analysis
- Extracted the ISO file using 7-Zip
- Found the file was mostly null bytes with a small embedded payload at offset 0x78a0
- Extracted 39 bytes and decoded using formula: `(index * 41) ^ byte ^ 76`
- Result: URL `http://loader.sweethearts.thm/cupid.ps1`

#### 5. Stage 3: PowerShell Obfuscation
- Downloaded `cupid.ps1` from the loader domain
- The script uses heavy obfuscation with character arrays and string manipulation
- Key sections:
  - `$h1` builds URL to `cdn.loveletters.thm/roses.jpg`
  - Function `F1` searches for marker `--VALENTINE_PAYLOAD_START-->`
  - Function `I1` performs XOR decryption with key "ROSES"
  - Downloads and executes VBScript payload

#### 6. Stage 4: Image Steganography
- Accessed `http://cdn.loveletters.thm/roses.jpg` with Windows User-Agent
- Found JPEG file containing hidden data
- Extracted payload by:
  - XORing file data (starting at offset 1) with key "ROSES"
  - Searching for Base64 string starting with "RGlt" (Base64 for "Dim")
  - Decoding the Base64 to reveal VBScript

#### 7. Stage 5: VBScript Dropper
The VBScript (`valentine.vbs`) performs:
- Creates objects for file system interaction, shell, and XMLHTTP
- Downloads `heartbeat.exe` from `http://cdn.loveletters.thm/heartbeat.exe`
- Saves it to `%APPDATA%\heartbeat.exe`
- Executes the downloaded executable

#### 8. Stage 6: Executable Analysis
- The `heartbeat.exe` is a Windows PE32+ executable
- Contains hardcoded C2 server: `api.valentinesforever.thm:8080`
- Uses Basic authentication with credentials: `cupid_agent:R0s3s4r3R3d!V10l3ts4r3Blu3#2024`
- Communicates with `/exfil` endpoint for command and control

#### 9. Data Exfiltration and Recovery
- Accessed `http://api.valentinesforever.thm:8080/exfil` with proper auth and User-Agent
- Retrieved list of encrypted files
- Downloaded each `.enc` file and sent it back to the same endpoint to decrypt
- One decrypted file (`61d07abe73c3.dec`) contained the flag

### Flag
**THM{l0v3_l3tt3r_fr0m_th3_90s_xoxo**

### Key Takeaways
1. Multi-stage payload delivery with each stage using different obfuscation techniques
2. Heavy reliance on Windows-specific User-Agent filtering to evade sandbox detection
3. Use of legitimate services (image hosting, file sharing) for C2 infrastructure
4. Strong encryption/XOR obfuscation at each stage requiring careful static analysis
5. Importance of tracking all network indicators and adding them to hosts file for analysis
6. The final payload exfiltrated sensitive data that could be recovered by mimicking the C2 protocol

### Tools Used
- curl (with custom headers/User-Agent)
- Python (for decoding and automation)
- 7-Zip (for ISO extraction)
- xxd/hexdump (for binary analysis)
- strings (for executable analysis)
- Burp Suite (for traffic analysis, though not explicitly shown in walkthrough)
