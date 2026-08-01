## Walkthrough: TryHackMe Room: "Room 404"

**Target IP:** `10.49.145.164`  
**Port:** `8080`  
**Difficulty:** Very Easy  
**Category:** Web / Information Disclosure

This room simulates a real-world scenario where a developer accidentally exposes the `.git` directory on a production server, leaking the entire source code history and sensitive data.

## 1. Initial Reconnaissance
Begin by scanning the target to identify open ports and services.
```bash
nmap -p- 10.49.145.164
```
**Results:**
- **Port 22/tcp**: SSH (Ignore for now; no credentials).
- **Port 8080/tcp**: HTTP (Web Application).

Navigate to `http://10.49.145.164:8080`. You will see the "Byte Lotus" hotel landing page. The site appears static with no obvious input fields or vulnerabilities.

## 2. Directory Enumeration
Since the homepage is static, perform directory brute-forcing to find hidden paths. Use `gobuster` or `dirb` with a standard wordlist.
```bash
gobuster dir -u http://10.49.145.164:8080 -w /usr/share/wordlists/dirb/common.txt
```
**Key Finding:**
The scan reveals a **`.git`** directory (Status 200).
- **Significance:** An exposed `.git` directory allows an attacker to download the entire repository, including commit history, source code, and potentially deleted files containing secrets.

## 3. Exploitation: Dumping the Repository
Manually browsing `.git` objects is complex because they are zlib-compressed. The professional approach is to use **`git-dumper`** to reconstruct the repository locally.

### Step A: Install `git-dumper`
If you encounter the `externally-managed-environment` error on Kali Linux, use the `--break-system-packages` flag.
```bash
# Clone the tool
git clone https://github.com/arthaud/git-dumper.git
cd git-dumper

# Install dependencies (Bypass PEP 668 restriction)
pip3 install -r requirements.txt --break-system-packages
```

### Step B: Execute the Dump
Point the tool at the exposed `.git` directory on the target and specify a local output folder.
```bash
python3 git_dumper.py http://10.49.145.164:8080/.git ./room404_dump
```
*Note: Ensure you include `/.git` in the URL.*

This command downloads all Git objects (commits, trees, blobs) and reconstructs the working directory in the `./room404_dump` folder.

## 4. Flag Recovery
Once the repository is dumped, navigate to the new directory and search for the flag format (`THM{...}`).
```bash
cd room404_dump
grep -r "THM{"
```
**Analysis:**
The search will highlight the flag inside the **`README.md`** file.
- **Context:** The flag was likely in a file that the developer deleted before pushing the final version of the website, but because it was committed to Git history previously, it remains recoverable in the object database.

**Final Flag:**
`THM{...................}`

## Summary of Lessons
1.  **Never expose `.git`**: Web servers (Apache/Nginx) must be configured to deny access to hidden directories like `.git`.
2.  **History is permanent**: Deleting a file in a later commit does not remove it from the Git history; sensitive data must be scrubbed using tools like `git filter-branch` or `BFG Repo-Cleaner`.
3.  **Automation**: Tools like `git-dumper` automate the complex process of manually decompressing Git objects, speeding up penetration tests.

