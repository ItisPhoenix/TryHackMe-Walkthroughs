# Task 7 - Signed Messages
## Love at First Breach 2026 - Advanced Track

**Room:** https://tryhackme.com/room/lafbctf2026-advanced

### Challenge Overview
Signed Messages is a secure messaging platform that claims to use RSA-2048 digital signatures for message authentication. The platform allows users to send secure messages and verify whether signatures are valid. However, there's a critical vulnerability in the key generation process.

**Target IP:** 10.49.144.4  
**Challenge Description:** Their messages are secret, unless you find the key.

---

## Step-by-Step Walkthrough

### Phase 1: Initial Reconnaissance

Scan the target to identify open ports:

```bash
nmap -sV -sC -p 5000,22 10.49.144.4
```

**Results:**
- Port 22/tcp: Open (SSH)
- Port 5000/tcp: Open (HTTP - Flask application)

Access the application:
```
http://10.49.144.4:5000
```

---

### Phase 2: Application Exploration

The Signed Messages platform provides:
- User registration/login
- Message signing and verification
- Claims to use RSA-2048 with RSA-PSS signatures

Create an account and explore the interface.

---

### Phase 3: Directory Enumeration

Discover hidden endpoints:

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.49.144.4:5000/FUZZ
```

**Key Discovery:** Hidden `/debug/` endpoint

Access:
```
http://10.49.144.4:5000/debug/
```

---

### Phase 4: Analyzing the Vulnerability

The debug page reveals critical information:

```
[2026-02-06 14:23:15] Development mode: ENABLED
[2026-02-06 14:23:15] Using deterministic key generation
[2026-02-06 14:23:15] Seed pattern: {username}_lovenote_2026_valentine
[2026-02-06 14:23:16] RSA modulus generated from p × q
[2026-02-06 14:23:16] RSA-2048 key pair successfully constructed
```

**Critical Findings:**
1. Deterministic key generation using seed: `{username}_lovenote_2026_valentine`
2. Keys are predictable - same username = same private key
3. Effective key size is ~512-bit (not 2048-bit as claimed)

---

### Phase 5: Exploitation - Private Key Reconstruction

Create the exploit script:

```python
import hashlib
from sympy import nextprime
from Crypto.PublicKey import RSA
from Crypto.Signature import pss
from Crypto.Hash import SHA256
from Crypto.Util.number import inverse

TARGET_USER = "admin"
TARGET_MESSAGE = "Hello"

def generate_admin_key():
    seed_str = f"{TARGET_USER}_lovenote_2026_valentine"
    seed_bytes = seed_str.encode()
    
    # Generate p
    sha256_p = hashlib.sha256(seed_bytes).hexdigest()
    p = nextprime(int(sha256_p, 16))
    
    # Generate q
    sha256_q = hashlib.sha256(seed_bytes + b"pki").hexdigest()
    q = nextprime(int(sha256_q, 16))
    
    n = p * q
    e = 65537
    phi = (p - 1) * (q - 1)
    d = inverse(e, phi)
    
    key = RSA.construct((n, e, d, p, q))
    
    modBits = key.size_in_bits()
    h = SHA256.new(TARGET_MESSAGE.encode())
    emLen = (modBits - 1 + 7) // 8
    maxSalt = emLen - h.digest_size - 2
    
    print(f"[*] Key size: {modBits} bits")
    
    if maxSalt < 0:
        raise ValueError("Key too small for PSS padding.")
    
    signer = pss.new(key, saltLen=maxSalt)
    signature = signer.sign(h)
    
    return signature.hex()

if __name__ == "__main__":
    signature = generate_admin_key()
    
    print("\n>>> SIGNATURE <<<")
    print(signature)
    print("\nVerify at /verify with:")
    print(f"User: {TARGET_USER}")
    print(f"Message: {TARGET_MESSAGE}")
```

Run the script to generate admin signature.

---

### Phase 6: Flag Acquisition

1. Run exploit script → get signature
2. Navigate to `/verify` page
3. Submit:
   - Username: `admin`
   - Message: (check the welcome message on site)
   - Signature: (generated signature)

**Flag:** `THM{PR3D1CT4BL3_S33D5_BR34K_H34RT5}`

---

## Key Takeaways

- **Deterministic crypto keys** completely break security
- Always verify cryptographic implementations
- Debug logs can reveal critical vulnerabilities

---

## Remediation

- Use cryptographically secure random number generators
- Never seed cryptographic operations with predictable data
- Follow proper key generation standards
