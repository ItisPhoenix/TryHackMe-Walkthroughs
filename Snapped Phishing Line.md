
# Snapped Phishing Line - TryHackMe Walkthrough

## Room Information
- **Link:** https://tryhackme.com/room/snappedphishingline
- **Category:** Phishing Analysis (SOC Level 1 Path)
- **Difficulty:** Easy
- **Target IP:** 10.49.169.239

## Scenario
You are an IT department personnel at SwiftSpend Financial. Several employees reported suspicious emails, and some have already submitted credentials and can no longer log in. Your task is to investigate the phishing campaign.

## Prerequisites
- Access to TryHackMe lab environment
- Basic knowledge of email analysis
- Familiarity with URL defanging
- VirusTotal/ThreatBook accounts (optional but helpful)

---

## Questions & Answers

### Q1: Who is the individual who received an email attachment containing a PDF?
**Answer:** `William McClean`

**Steps:**
1. Navigate to the Desktop folder:
   ```bash
   cd /home/damianhall/Desktop/phish-emails/
   ```
2. List all email files:
   ```bash
   ls -la
   ```
3. Search for emails containing PDF attachments:
   ```bash
   grep -l -i "pdf" *.eml
   ```
4. Open the identified email file in Thunderbird or text editor
5. Check the "To:" field to find the recipient

**Explanation:** Among the 5 emails, only one contains a PDF attachment. The recipient of that specific email is William McClean.

---

### Q2: What email address was used by the adversary to send the phishing emails?
**Answer:** `Accounts.Payable@groupmarketingonline.icu`

**Steps:**
1. Open each email file and examine the "From:" header:
   ```bash
   grep -i "^from:" *.eml
   ```
2. Or view emails in Thunderbird for better readability
3. Identify the common sender address across phishing emails

**Explanation:** The attacker used this spoofed email address that appears to be from an accounts payable department to make the emails seem legitimate.

---

### Q3: What is the redirection URL to the phishing page for Zoe Duncan? (Defanged Format)
**Answer:** `hxxp[://]kennaroads[.]buzz/data/Update365/office365/40e7baa2f826a57fcf04e5202526f8bd/?email=zoe[.]duncan@swiftspend[.]f`

**Steps:**
1. Find the email sent to Zoe Duncan
2. Download or view the HTML attachment
3. View the HTML source code to find the malicious URL
4. Use CyberChef to defang the URL:
   - Go to https://gchq.github.io/CyberChef/
   - Add "Escape dots" operation
   - Add "Escape http" operation

**Defanging Explained:**
- `http` → `hxxp`
- `.` → `[.]`
- `://` → `[://]`

---

### Q4: What is the URL to the ZIP archive of the phishing kit? (Defanged Format)
**Answer:** `hxxp[://]kennaroads[.]buzz/data/Update365[.]zip`

**Steps:**
1. Visit the base URL: `http://kennaroads.buzz/`
2. Enumerate directories manually:
   - `/data/` - Contains the ZIP archive
   - `/Update365/` - Contains log files
   - `/Update365/office365/` - Contains phishing kit files
3. Identify the ZIP file at `/data/Update365.zip`
4. Defang the URL using CyberChef

---

### Q5: What is the SHA256 hash of the phishing kit archive?
**Answer:** `ba3c15267393419eb08c7b2652b8b6b39b406ef300ae8a18fee4d16b19ac9686`

**Steps:**
1. Download the ZIP file:
   ```bash
   wget http://kennaroads.buzz/data/Update365.zip
   ```
2. Calculate SHA256 hash:
   ```bash
   sha256sum Update365.zip
   ```
3. Copy the resulting hash

---

### Q6: When was the phishing kit archive first submitted? (Format: YYYY-MM-DD HH:MM:SS UTC)
**Answer:** `2020-04-08 05:42:39 UTC`

**Steps:**
1. Go to VirusTotal: https://www.virustotal.com
2. Search by the SHA256 hash from Q5
3. Navigate to the "Details" tab
4. Look for "History" or "First Submission" date
5. If not available, check the "Relations" tab

---

### Q7: When was the SSL certificate the phishing domain used to host the phishing kit archive first logged? (Format: YYYY-MM-DD)
**Answer:** `2020-06-25`

**Note:** The SSL certificate is no longer available for reliable verification. TryHackMe provides the answer in the hint.

**Steps (if certificate was available):**
1. Search the domain on VirusTotal
2. Go to "Relations" tab → "Historical SSL Certificates"
3. Find the first logged certificate date

Alternative method using ThreatBook:
1. Go to https://threatbook.io
2. Search for `kennaroads.buzz`
3. Check SSL certificate history

---

### Q8: What was the email address of the user who submitted their password twice?
**Answer:** `michael.beckhoff@swiftspend.f`

**Steps:**
1. Navigate to the log file:
   ```bash
   wget http://kennaroads.buzz/data/Update365/log.txt
   ```
2. Search for email addresses and count occurrences:
   ```bash
   grep -i "email" log.txt | sort | uniq -c
   ```
3. Find the email that appears twice
4. Alternatively:
   ```bash
   grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]+" log.txt | sort | uniq -c
   ```

---

### Q9: What was the email address used by the adversary to collect compromised credentials?
**Answer:** `jamestanner2299@gmail.com`

**Steps:**
1. Extract the phishing kit:
   ```bash
   unzip Update365.zip
   ```
2. Navigate to the office365 directory
3. Search for credential collection logic:
   ```bash
   grep -r "@" .
   grep -r "mail" *.php
   grep -r "password" *.php
   ```
4. Look for email sending functions in PHP files (typically in `enterpassword.php` or validation scripts)

---

### Q10: The adversary used other email addresses in the obtained phishing kit. What is the email address that ends in "@gmail.com"?
**Answer:** `jamestanner2299@gmail.com`

**Steps:**
1. Search for Gmail addresses in the extracted phishing kit:
   ```bash
   grep -r "@gmail.com" .
   grep -ri "gmail" .
   ```
2. Look in PHP files, especially validation scripts

---

### Q11: What is the hidden flag?
**Answer:** `THM{pL4y_w1Th_tH3_URL}`

**Steps:**
1. Enumerate the `/Update365/office365/` directory for hidden files
2. Try accessing `flag.txt`:
   ```
   http://kennaroads.buzz/data/Update365/office365/flag.txt
   ```
3. The flag is Base64 encoded and reversed:
   ```
   fUxSVV8zSHRfaFQxd195NExwe01IVAo=
   ```
4. Decode with CyberChef:
   - Add "From Base64" operation
   - Add "Reverse" operation

5. Or use command line:
   ```bash
   echo "fUxSVV8zSHRfaFQxd195NExwe01IVAo=" | base64 -d | rev
   ```

**Decoding Process:**
```
Original: fUxSVV8zSHRfaFQxd195NExwe01IVAo=
Base64 decoded: oIHM1pwx0N41xdQfRHSz8VVRxu
Reversed: THM{pL4y_w1Th_tH3_URL}
```

---

## Key Learnings

### Phishing Analysis Techniques
1. **Email Header Analysis** - Understanding From, To, Reply-To fields
2. **URL Defanging** - Making malicious URLs safe for sharing
3. **Log Analysis** - Using grep, sort, uniq to identify patterns
4. **OSINT Tools** - VirusTotal, ThreatBook for threat intelligence
5. **Source Code Review** - Analyzing PHP files for malicious functionality

### Tools Used
| Tool | Purpose |
|------|---------|
| Thunderbird | Email viewing and analysis |
| CyberChef | URL defanging, Base64 decoding |
| VirusTotal | File/URL reputation, submission dates |
| ThreatBook | Domain intelligence, SSL certificates |
| grep | Log and source code searching |
| sha256sum | File hashing |

### URL Defanging Reference
| Original | Defanged |
|----------|----------|
| `http://` | `hxxp[://]` |
| `https://` | `hxxps[://]` |
| `.` | `[.]` |

---

## Troubleshooting

### Issue: Cannot connect to machine
- Wait for the machine to fully initialize (2-3 minutes)
- Refresh the connection
- Check if VPN is properly connected

### Issue: SSL certificate question answer not working
- The certificate is no longer available
- Use the provided answer: `2020-06-25`

### Issue: Flag decoding not working
- Ensure you're using "Reverse" operation (not "Rotate")
- Check that Base64 decoding is done first, then reverse

---
