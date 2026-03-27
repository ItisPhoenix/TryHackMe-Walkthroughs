# The Case: Seven Minutes on the Seine - Walkthrough

**Room Link:** [The Case: Seven Minutes on the Seine](https://tryhackme.com/room/thecasesevenminutesontheseine)  
**Target IP:** `10.48.130.7`  
**Difficulty:** Medium/OSINT  
**Author:** Gemini CLI

## 1. Reconnaissance

### Nmap Scan
First, we perform a service discovery scan on the target IP:

```bash
nmap -sV -sC -T4 --open 10.48.130.7
```

**Results:**
- **Port 22 (SSH):** OpenSSH 8.9p1 Ubuntu
- **Port 80 (HTTP):** Werkzeug httpd 3.0.1 (Python 3.11.14)
  - Title: "Investigation Notes - Detective Case File"

The HTTP service is a Flask/Werkzeug application that presents an "Investigation Report" interface.

## 2. Web Application Analysis

Visiting `http://10.48.130.7/` reveals a case file interface with 5 questions that need to be answered to close the case and retrieve the flag.

The questions are:
1. **The Way In**: Museum entrance name on the river side and its official closure date.
2. **The Queen's Reliquary Brooch**: Inventory number, maker, acquisition mode/year, and status.
3. **International Alert**: INTERPOL reference IDs for the stolen pieces.
4. **Ceiling Clue**: Details of the central ceiling painting in the Galerie d'Apollon.
5. **Bridge to Nowhere**: Bridge directly south of the entrance.

## 3. OSINT Investigation

### Q1: The Way In
Researching the "Louvre heist" and "ladder truck" leads to a specific scenario involving the **Porte des Lions** entrance.
- **Entrance:** `PORTE_DES_LIONS`
- **Closure Date:** `22_OCT_2024`
- **Answer Format:** `PORTE_DES_LIONS-22_OCT_2024`

### Q2: The Queen's Reliquary Brooch
The item is the **Reliquary Brooch of Empress Eugénie** (Grand Noeud de Corsage).
- **Inventory Number:** `MV1024` (Note: In this specific challenge context, the `MV` prefix is used).
- **Maker:** `BAPST` (Paul-Alfred Bapst)
- **Acquisition:** `AFFECTE` (Assigned in 1887)
- **Status:** `NON_EXPOSE` (Not on display)
- **Answer Format:** `MV1024-BAPST-AFFECTE-1887-NON_EXPOSE`

### Q3: International Alert
Two pieces were flagged with INTERPOL reference IDs.
- **Sapphire Diadem:** `2025/359.1`
- **Reliquary Brooch:** `2025/359.5`
- **Answer Format:** `2025/359.1,2025/359.5`

### Q4: Ceiling Clue
The central ceiling painting in the Galerie d'Apollon is **Apollo Slaying the Python** by Eugène Delacroix.
- **Title:** `APOLLON_VAINQUEUR`
- **Inventory:** `INV_3818`
- **Dimensions:** `8mx7.5m`
- **Answer Format:** `APOLLON_VAINQUEUR-INV_3818-8mx7.5m`

### Q5: Bridge to Nowhere
The **Porte des Lions** entrance is located on the Quai François Mitterrand. The bridge directly south of this location is the **Pont Royal**.
- **Answer:** `PONT_ROYAL`

## 4. Retrieving the Flag

After submitting all five correct answers to the backend, we receive the final report confirmation.

**Flag:** `THM{n1c3_h31st_r3s34rch}`

---

# Task 2: Louvre Protocol - Walkthrough

**Target IP:** `10.48.167.228`  
**Difficulty:** Easy/Medium

## 1. Reconnaissance

### Nmap Scan
Performing a service discovery scan on the second target IP:

```bash
nmap -sV -sC -T4 --open 10.48.167.228
```

**Results:**
- **Port 22 (SSH):** OpenSSH 8.9p1 Ubuntu
- **Port 80 (HTTP):** Werkzeug httpd 3.0.1 (Python 3.11.14)
  - Title: "Louvre Security System - Login"

The HTTP service is a login portal for the Louvre Museum Security Monitoring System.

## 2. Authentication

The login page at `http://10.48.167.228/login` requires a username and password. Based on common security defaults or potential clues from the previous investigation, we attempt:

- **Username:** `louvre`
- **Password:** `louvre`

The login is successful, and we are redirected to the CCTV monitoring dashboard.

## 3. Investigating the Heist

The dashboard allows viewing CCTV footage by date. From Task 1, we know the heist occurred on **October 19, 2025**.

1.  Navigate to the CCTV view for the date: `http://10.48.167.228/cctv/2025-10-19`.
2.  Upon loading the footage for the date of the heist, the system triggers a critical alert banner.

## 4. Retrieving the Flag

The alert banner displayed on the dashboard reveals the security flag for this task.

**Flag:** `THM{cctv_4ud1ts_4r3_fun}`
