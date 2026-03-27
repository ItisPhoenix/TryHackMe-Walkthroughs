# Warzone 1 - TryHackMe Walkthrough

**Room Link:** [https://tryhackme.com/room/warzoneone](https://tryhackme.com/room/warzoneone)
**Difficulty:** Medium
**Target Machine IP:** 10.48.146.92
**Date Completed:** 2026-03-23

## Scenario
You work as a Tier 1 Security Analyst L1 for a Managed Security Service Provider (MSSP). Today you’re tasked with monitoring network alerts. A few minutes into your shift, you get your first network case: Potentially Bad Traffic and Malware Command and Control Activity detected. Your race against the clock starts. Inspect the PCAP and retrieve the artifacts to confirm this alert is a true positive.

## Tools Used
- **Brim (Zui)** - For fast PCAP searching and Suricata alert analysis
- **Wireshark** - For deep packet analysis and HTTP stream inspection
- **VirusTotal** - For threat intelligence enrichment
- **NetworkMiner** - For file extraction and artifact analysis

## Walkthrough

### Step 1: Load the PCAP in Brim
1. Navigate to the Tools folder on the provided VM
2. Launch Brim application
3. Drag and drop the `zone1.pcap` file from the Desktop into Brim

### Step 2: Identify Alert Signature (Question 1.1)
**Question:** What was the alert signature for "Malware Command and Control Activity Detected"?

**Steps:**
1. In Brim, go to the Queries tab on the left
2. Scroll down and double-click on "Suricata Alerts by Category"
3. Look for the alert category "Malware Command and Control Activity Detected"
4. To get the exact signature, modify the query:
   ```bash
   event_type=="alert" | count() by alert.severity,alert.category,alert.signature | sort count
   ```
5. Identify the signature under the malware category

**Answer:** `ET MALWARE MirrorBlast CnC Activity M3`

### Step 3: Identify Source IP Address (Question 1.2)
**Question:** What is the source IP address? Enter your answer in a defanged format.

**Steps:**
1. Use the "Unique Network Connections" query in Brim:
   ```bash
   _path=="conn" | cut id.orig_h, id.resp_p, id.resp_h | sort | uniq
   ```
2. Or look at the alert details for the source IP
3. Defang the IP address (replace dots with `[.]`)

**Answer:** `172[.]16[.]1[.]102`

### Step 4: Identify Destination IP Address (Question 1.3)
**Question:** What IP address was the destination IP in the alert? Enter your answer in a defanged format.

**Steps:**
1. Use the "Suricata Alerts by Source and Destination" query:
   ```bash
   event_type=="alert" | alerts := union(alert.category) by src_ip, dest_ip
   ```
2. Filter for the malware alert category
3. Defang the destination IP address

**Answer:** `169[.]239[.]128[.]11`

### Step 5: VirusTotal Analysis - Domain (Question 1.4)
**Question:** Inspect the IP address in VirusTotal. Under Relations > Passive DNS Replication, which domain has the most detections?

**Steps:**
1. Go to [VirusTotal](https://www.virustotal.com)
2. Search for the destination IP: `169.239.128.11`
3. Navigate to the "Relations" tab
4. Look at "Passive DNS Replication" section
5. Identify the domain with the highest detection count
6. Defang the domain

**Answer:** `fidufagios[.]com`

### Step 6: VirusTotal - Threat Group (Question 1.5)
**Question:** Still in VirusTotal, under Community, what threat group is attributed to this IP address?

**Steps:**
1. In VirusTotal, with the same IP selected
2. Navigate to the "Community" tab
3. Look for threat group attributions
4. Search online for "TA505" to understand the threat actor

**Answer:** `TA505`

### Step 7: Malware Family (Question 1.6)
**Question:** What is the malware family?

**Steps:**
1. The alert signature already gives us a clue: "MirrorBlast"
2. Confirm with VirusTotal analysis of associated files
3. Search for "MirrorBlast malware" for additional context

**Answer:** `MirrorBlast`

### Step 8: File Type Analysis (Question 1.7)
**Question:** What was the majority file type listed under Communicating Files?

**Steps:**
1. In VirusTotal, still analyzing the destination IP
2. Navigate to "Communicating Files" section
3. Look at the file types listed
4. Identify the most common file type

**Answer:** `Windows Installer` (.msi files)

### Step 9: User-Agent Analysis (Question 1.8)
**Question:** Inspect the web traffic for the flagged IP address; what is the user-agent in the traffic?

**Steps:**
1. Back in Brim, use the HTTP traffic query:
   ```bash
   _path=="http" | cut id.orig_h, id.resp_h, id.resp_p, method,host, uri, user_agent
   ```
2. Filter for traffic to/from the malicious IP
3. Alternatively, in Wireshark:
   - Open the PCAP file
   - Search for the malicious IP
   - Right-click a packet → Follow → HTTP Stream
   - Look for the User-Agent header

**Answer:** `REBOL View 2.7.8.3.1`

### Step 10: Identify Additional IPs (Question 1.9)
**Question:** Retrace the attack; there were multiple IP addresses associated with this attack. What were two other IP addresses? Enter the IP addresses defanged and in numerical order.

**Steps:**
1. In Brim, use the query to find HTTP connections:
   ```bash
   _path=="http" | cut id.orig_h, id.resp_h, host | uniq -c
   ```
2. Identify unique destination IPs
3. Verify each in VirusTotal to confirm malicious activity
4. Select the two IPs in numerical order
5. Defang both IPs

**Answer:** `185[.]10[.]68[.]235,192[.]36[.]27[.]92`

### Step 11: Downloaded File Names (Question 1.10)
**Question:** What were the file names of the downloaded files? Enter the answer in the order to the IP addresses from the previous question.

**Steps:**
1. In Brim, use the "File Activity" filter
2. Look for .msi file downloads
3. Alternatively, in Wireshark:
   - Go to File → Export Objects → HTTP
   - Look for .msi files
4. Match each file to its source IP

**Answer:** `filter.msi,10opd3r_load.msi`

### Step 12: First File Paths (Question 1.11)
**Question:** Inspect the traffic for the first downloaded file. Two files will be saved to the same directory. What is the full file path of the directory and the name of the two files?

**Steps:**
1. In Wireshark, search for `filter.msi`
2. Right-click a packet → Follow → HTTP Stream
3. Look for file paths in the stream data
4. The paths will be in the format: `C:\ProgramData\...`

**Answer:** `C:\ProgramData\001\arab.bin,C:\ProgramData\001\Action1_arab.exe`

### Step 13: Second File Paths (Question 1.12)
**Question:** Inspect the traffic from the second downloaded file. Two files will be saved to the same directory. What is the full file path of the directory and the name of the two files?

**Steps:**
1. In Wireshark, search for `10opd3r_load.msi`
2. Right-click a packet → Follow → HTTP Stream
3. Scroll through the stream to find file paths
4. Look for executable and script files

**Answer:** `C:\ProgramData\Local\Google\rebol-view-278-3-1.exe,C:\ProgramData\Local\Google\exemple.rb`

## Summary of Answers

| Question | Answer |
|----------|--------|
| 1.1 Alert signature | ET MALWARE MirrorBlast CnC Activity M3 |
| 1.2 Source IP | 172[.]16[.]1[.]102 |
| 1.3 Destination IP | 169[.]239[.]128[.]11 |
| 1.4 Domain with most detections | fidufagios[.]com |
| 1.5 Threat group | TA505 |
| 1.6 Malware family | MirrorBlast |
| 1.7 Majority file type | Windows Installer |
| 1.8 User-Agent | REBOL View 2.7.8.3.1 |
| 1.9 Two other IPs | 185[.]10[.]68[.]235,192[.]36[.]27[.]92 |
| 1.10 Downloaded files | filter.msi,10opd3r_load.msi |
| 1.11 First file paths | C:\ProgramData\001\arab.bin,C:\ProgramData\001\Action1_arab.exe |
| 1.12 Second file paths | C:\ProgramData\Local\Google\rebol-view-278-3-1.exe,C:\ProgramData\Local\Google\exemple.rb |

## Key Learnings
1. **Suricata Alert Analysis**: Using Brim to quickly filter and analyze IDS/IPS alerts
2. **PCAP Analysis**: Combining Brim for quick searches with Wireshark for deep packet inspection
3. **Threat Intelligence**: Using VirusTotal to enrich findings with domain reputation, threat groups, and file analysis
4. **Attack Chain Reconstruction**: Tracing multiple IPs and file downloads to understand the full attack
5. **File Path Extraction**: Using HTTP stream following to identify dropped file locations

## Conclusion
This room provided excellent hands-on experience with network forensics and incident response. By using Brim for quick alert triage, Wireshark for detailed packet analysis, and VirusTotal for threat intelligence, we successfully confirmed the IDS alert as a true positive. The attack involved MirrorBlast malware distributed via MSI installers, with command and control communications to TA505 infrastructure.

---
