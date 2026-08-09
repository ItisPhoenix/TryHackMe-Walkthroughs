## Walkthrough: TryHackMe Room: "Management Wants a Word"

> **Room:** Management Wants a Word  
> **Platform:** TryHackMe — Hacker Holidays 2026  
> **Category:** Forensics / Windows / Cryptography  
> **Difficulty:** Hard  
> **Points:** 120

---

## Overview

The challenge provides a Windows forensic triage image from a laptop belonging to a user named **Vera**.

The objective is to investigate artifacts across the system, recover a hidden password, use it to unlock an encrypted VeraCrypt container, and ultimately recover the flag.

The investigation chain is:

```text
Windows Registry Hives
        |
        v
LSA DefaultPassword
        |
        v
Vera's DPAPI Master Key
        |
        v
Chrome AES Encryption Key
        |
        v
Chrome Login Data
        |
        v
SecureVault Password
        |
        v
VeraCrypt Container
        |
        v
Secret Documents
        |
        v
Invoice PDF
        |
        v
Manual Image Inspection
        |
        v
Flag
```

---

# 1. Initial Evidence

The extracted KAPE evidence is located under:

```text
KAPE/
```

The main suspicious file is:

```text
KAPE/C/Users/vera/Documents/backup
```

Check the file type:

```bash
file KAPE/C/Users/vera/Documents/backup
```

Output:

```text
KAPE/C/Users/vera/Documents/backup: data
```

The file is 100 MiB and does not have a recognizable archive or filesystem signature.

Inspect the first 64 bytes:

```bash
xxd -l 64 KAPE/C/Users/vera/Documents/backup
```

Output:

```text
00000000: f372 f7cc d607 4b17 a8aa 8865 12af abdf  .r....K....e....
00000010: f293 9a74 72ea acbc bee5 b479 4c88 5c7f  ...tr......yL.\
00000020: 5f53 fc44 2988 f8fc ae98 21dd c26a 2a9b  _S.D).....!..j*.
00000030: 8c45 8f73 8ac4 1cdc 8377 bb46 9636 4807  .E.s.....w.F.6H.
```

The data appears high-entropy, which is consistent with an encrypted container.

The challenge also provides the clue:

```text
1.26.29
```

This points toward **VeraCrypt**, but we need to recover the password before attempting to open the container.

---

# 2. Locate Chrome Artifacts

The challenge contains the clue:

> "a browser will remember things for you that you never told anyone else"

This strongly suggests investigating Chrome.

Find the relevant artifacts:

```bash
find KAPE/C/Users/vera -type f \
  \( -iname 'Login Data' \
  -o -iname 'Local State' \
  -o -iname 'Web Data' \
  -o -iname 'History' \) -print
```

The important files are:

```text
KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Local State
KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data
KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/History
KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Web Data
```

### Artifact purposes

| Artifact | Purpose |
|---|---|
| `History` | Browser activity |
| `Login Data` | Saved usernames and encrypted passwords |
| `Web Data` | Autofill information |
| `Local State` | Chrome encryption key |

---

# 3. Examine Chrome Saved Credentials

Query the `Login Data` database:

```bash
sqlite3 "KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data" \
"SELECT origin_url, username_value, length(password_value) FROM logins;"
```

Result:

```text
http://bytelotus.thm:8080/|VeraSecretVault|56
```

This gives us an important lead:

```text
URL:      http://bytelotus.thm:8080/
Username: VeraSecretVault
Password: Encrypted
```

The 56-byte value is the encrypted password blob, not the plaintext password length.

Extract the encrypted password:

```bash
sqlite3 "KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data" \
"SELECT hex(password_value) FROM logins WHERE origin_url='http://bytelotus.thm:8080/';"
```

The blob begins with:

```text
763130
```

which is:

```text
v10
```

This is Chrome's encrypted credential format.

---

# 4. Locate Vera's DPAPI Master Key

Chrome's password is protected using an AES key, and that key is itself protected by Windows DPAPI.

Locate Vera's DPAPI master keys:

```bash
find KAPE/C/Users/vera/AppData/Roaming/Microsoft/Protect \
  -type f -printf '%f %p\n'
```

The relevant SID is:

```text
S-1-5-21-2529683458-431225740-1723070931-1000
```

The relevant master-key GUID is:

```text
c90719ef-5b98-474e-b934-136d606a702a
```

The master-key file is:

```text
KAPE/C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a
```

---

# 5. Extract Windows LSA Secrets

The triage contains the necessary registry hives:

```text
KAPE/C/Windows/System32/config/SAM
KAPE/C/Windows/System32/config/SYSTEM
KAPE/C/Windows/System32/config/SECURITY
```

Use Impacket:

```bash
impacket-secretsdump \
  -sam KAPE/C/Windows/System32/config/SAM \
  -system KAPE/C/Windows/System32/config/SYSTEM \
  -security KAPE/C/Windows/System32/config/SECURITY \
  LOCAL
```

The important output is:

```text
[*] Dumping LSA Secrets
[*] DefaultPassword
(Unknown User):minivera
```

Therefore, Vera's relevant Windows password is:

```text
minivera
```

This is the password required to decrypt the DPAPI master key.

---

# 6. Decrypt Vera's DPAPI Master Key

Use the recovered password:

```bash
impacket-dpapi masterkey \
  -file "KAPE/C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a" \
  -sid "S-1-5-21-2529683458-431225740-1723070931-1000" \
  -password "minivera"
```

The important output is:

```text
Decrypted key with User Key (SHA1)
Decrypted key: 0x5e5715ec9b6df5a86e97902692a66d28e691f05d5bc1e04d0159cfe960e94c978c07e5004a0179d3a96df2468885a28175b0b02cc064445f116a752d2b3e9d40
```

The recovered DPAPI master key is:

```text
5e5715ec9b6df5a86e97902692a66d28e691f05d5bc1e04d0159cfe960e94c978c07e5004a0179d3a96df2468885a28175b0b02cc064445f116a752d2b3e9d40
```

---

# 7. Extract Chrome's DPAPI-Protected Key

Set the `Local State` path:

```bash
LOCAL_STATE="KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Local State"
```

Extract and decode Chrome's encrypted key:

```bash
jq -r '.os_crypt.encrypted_key' "$LOCAL_STATE" |
base64 -d |
xxd
```

The decoded value begins with:

```text
4450415049
```

which represents:

```text
DPAPI
```

Chrome therefore stores:

```text
DPAPI + encrypted key blob
```

Remove the five-byte `DPAPI` prefix:

```bash
jq -r '.os_crypt.encrypted_key' "$LOCAL_STATE" |
base64 -d |
tail -c +6 > chrome-key.dpapi
```

Verify:

```bash
xxd -l 16 chrome-key.dpapi
```

Expected beginning:

```text
01000000 d08c9ddf 0115d111 8c7a00c0
```

---

# 8. Decrypt Chrome's AES Key

Set the recovered DPAPI master key:

```bash
export MASTERKEY='5e5715ec9b6df5a86e97902692a66d28e691f05d5bc1e04d0159cfe960e94c978c07e5004a0179d3a96df2468885a28175b0b02cc064445f116a752d2b3e9d40'
```

Decrypt the Chrome DPAPI blob:

```bash
python3 - "$MASTERKEY" <<'PY'
import sys
from impacket.dpapi import DPAPI_BLOB

masterkey = bytes.fromhex(sys.argv[1])

with open("chrome-key.dpapi", "rb") as f:
    blob = DPAPI_BLOB(f.read())

decrypted = blob.decrypt(masterkey)

if decrypted is None:
    raise SystemExit("DPAPI decryption failed")

with open("chrome-aes.key", "wb") as f:
    f.write(decrypted)

print(f"Wrote {len(decrypted)} bytes")
print(f"Chrome AES key: {decrypted.hex()}")
PY
```

The recovered Chrome AES key is:

```text
206a39a0971327ea9487e4aea9844f5d3670162456982276939a712646da0b02
```

It is 32 bytes long, corresponding to AES-256.

---

# 9. Decrypt Chrome's Saved Password

Set the database path:

```bash
LOGIN_DATA="KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data"
```

Decrypt the saved credential:

```bash
python3 - "$LOGIN_DATA" ./chrome-aes.key <<'PY'
import sqlite3
import sys
from pathlib import Path
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

database = Path(sys.argv[1]).resolve()
keyfile = Path(sys.argv[2]).resolve()

key = keyfile.read_bytes()

if len(key) != 32:
    raise SystemExit(f"Unexpected AES key length: {len(key)}")

db = sqlite3.connect(database.as_uri() + "?mode=ro", uri=True)

for url, username, encrypted in db.execute("""
    SELECT origin_url, username_value, password_value
    FROM logins
"""):
    blob = bytes(encrypted)

    if not blob.startswith((b"v10", b"v11")):
        continue

    nonce = blob[3:15]
    ciphertext_and_tag = blob[15:]

    password = AESGCM(key).decrypt(
        nonce,
        ciphertext_and_tag,
        None
    ).decode("utf-8", errors="replace")

    print(f"URL:      {url}")
    print(f"Username: {username}")
    print(f"Password: {password}")
PY
```

Result:

```text
Username: VeraSecretVault
Password: Wh4t1sV3raD0inG0nTh1sH0st
```

We now have the password for the encrypted backup.

---

# 10. Open the VeraCrypt Container

Kali did not have the `veracrypt` command available, but `cryptsetup` supports VeraCrypt/TCRYPT containers.

Run:

```bash
sudo cryptsetup tcryptOpen \
  --veracrypt \
  KAPE/C/Users/vera/Documents/backup \
  vera_backup
```

Enter:

```text
Wh4t1sV3raD0inG0nTh1sH0st
```

Verify the mapper:

```bash
ls -l /dev/mapper/vera_backup
```

Expected:

```text
/dev/mapper/vera_backup -> ../dm-0
```

Check the filesystem:

```bash
sudo blkid /dev/mapper/vera_backup
```

The filesystem is:

```text
TYPE="vfat"
```

---

# 11. Mount the Container

Create a mount point:

```bash
sudo mkdir -p /mnt/vera
```

Mount read-only:

```bash
sudo mount -o ro /dev/mapper/vera_backup /mnt/vera
```

Inspect the contents:

```bash
ls -la /mnt/vera
```

We find:

```text
$RECYCLE.BIN
secret_financial_documents
System Volume Information
```

The interesting directory is:

```text
/mnt/vera/secret_financial_documents
```

---

# 12. Inspect the Secret Documents

Run:

```bash
ls -lah /mnt/vera/secret_financial_documents
```

Contents:

```text
important_invoice_byte_lotus.pdf
transactions_q3.csv
```

The PDF is the relevant artifact for the final stage.

---

# 13. Analyze the PDF

Attempt normal text extraction:

```bash
pdftotext \
  /mnt/vera/secret_financial_documents/important_invoice_byte_lotus.pdf -
```

No useful text is returned.

Check the PDF:

```bash
pdfinfo \
  /mnt/vera/secret_financial_documents/important_invoice_byte_lotus.pdf
```

Important properties:

```text
Pages:       1
Encrypted:   no
PDF version: 1.7
File size:   26747 bytes
```

There is no useful metadata.

---

# 14. Identify Embedded Images

Run:

```bash
pdfimages -list \
  /mnt/vera/secret_financial_documents/important_invoice_byte_lotus.pdf
```

The PDF contains a:

```text
636 x 724
```

image and a corresponding transparency mask.

Therefore, the invoice is image-based rather than text-based.

---

# 15. Extract the Invoice Image

Extract the images:

```bash
pdfimages -png \
  /mnt/vera/secret_financial_documents/important_invoice_byte_lotus.pdf \
  /tmp/invoice
```

Check the extracted files:

```bash
ls -lh /tmp/invoice*
```

Output:

```text
/tmp/invoice-000.png
/tmp/invoice-001.png
```

The first image is the actual invoice.

---

# 16. OCR and Manual Verification

OCR can be attempted:

```bash
tesseract /tmp/invoice-000.png stdout
```

However, OCR incorrectly interprets some of the stylized characters in the flag.

For example, OCR may interpret:

```text
v3r4
```

incorrectly, and similarly confuse:

```text
Al0ng
```

with:

```text
Along
```

Therefore, the extracted image must be **visually inspected manually**.

Manual inspection reveals the actual flag:

```text
THM{..................}
```

---

# 17. Final Flag

```text
THM{...................}
```

---

# 18. Complete Attack Chain

```text
backup
  |
  | 100 MiB encrypted data
  v
Suspected VeraCrypt container
  |
  v
Chrome Login Data
  |
  | saved credential for SecureVault
  v
VeraSecretVault
  |
  | encrypted using Chrome v10
  v
Chrome Local State
  |
  | DPAPI protected
  v
Windows DPAPI
  |
  | requires Vera's password
  v
SECURITY / SYSTEM / SAM
  |
  v
LSA DefaultPassword
  |
  v
minivera
  |
  v
DPAPI Master Key
  |
  v
Chrome AES-256 Key
  |
  v
Decrypt Chrome Login Data
  |
  v
Wh4t1sV3raD0inG0nTh1sH0st
  |
  v
VeraCrypt backup
  |
  v
secret_financial_documents
  |
  v
important_invoice_byte_lotus.pdf
  |
  v
Embedded invoice image
  |
  v
Manual visual inspection
  |
  v
THM{....................}
```

---

# 19. Key Findings

| Artifact | Finding |
|---|---|
| `Documents/backup` | 100 MiB encrypted container |
| Chrome `Login Data` | Saved SecureVault credential |
| SecureVault username | `VeraSecretVault` |
| LSA `DefaultPassword` | `minivera` |
| DPAPI Master Key GUID | `c90719ef-5b98-474e-b934-136d606a702a` |
| Chrome AES key | `206a39a0971327ea9487e4aea9844f5d3670162456982276939a712646da0b02` |
| SecureVault password | `Wh4t1sV3raD0inG0nTh1sH0st` |
| Container filesystem | VFAT |
| Hidden directory | `secret_financial_documents` |
| Final document | `important_invoice_byte_lotus.pdf` |
| Final flag | `THM{...............}` |

---

# 20. Lessons Learned

### Chrome forensics

Chrome's `Login Data` database does not normally expose saved passwords directly. The password is encrypted and requires the browser's encryption key.

Important artifacts include:

```text
Login Data
Local State
Web Data
History
```

### Windows DPAPI

Windows DPAPI protects secrets using user/system-derived cryptographic material.

In this challenge:

```text
LSA DefaultPassword
        |
        v
minivera
        |
        v
DPAPI Master Key
        |
        v
Chrome encryption key
```

### VeraCrypt

The 100 MiB `backup` file initially appears as:

```text
backup: data
```

because encrypted container data does not expose a conventional filesystem signature.

The password recovered from Chrome unlocks the container:

```text
Wh4t1sV3raD0inG0nTh1sH0st
```

### PDF Forensics

The final PDF contained no useful text layer. It consisted primarily of an embedded image.

Therefore:

```text
PDF
 |
 v
Embedded image
 |
 v
Manual visual inspection
```

was necessary.

### OCR Limitations

OCR was useful for confirming the invoice contents, but it was **not reliable enough to determine the exact flag**.

The final flag was verified by manually inspecting the extracted image.

---

# 21. Cleanup

After completing the investigation, unmount the volume:

```bash
sudo umount /mnt/vera
```

Then close the device mapper:

```bash
sudo cryptsetup close vera_backup
```

Verify:

```bash
ls /dev/mapper/vera_backup
```

The mapper should no longer exist.

---

## Final Flag

```text
THM{...................}
```
