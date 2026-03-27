# Dark Matter Walkthrough (TryHackMe)

## 1. Scenario
Hackfinitiy high school has been hit by **DarkInjector's ransomware**, which encrypted critical files. Debugging data was left behind in the `/tmp` directory of the target machine, which we will use to recover the RSA private key and decrypt the files.

## 2. Reconnaissance & Enumeration
Initial scanning was performed on the target IP `10.48.150.164`.

**Nmap Scan:**
- **22/tcp**: SSH
- **80/tcp**: WebSockify Python/3.12.3

The presence of WebSockify suggests a VNC or remote desktop interface for the challenge.

## 3. Cryptography Analysis
The clue indicates that debugging data is saved in `/tmp`. Based on the challenge context, we find the RSA public key configuration:

- **Modulus (n):** `340282366920938460843936948965011886881`
- **Public Exponent (e):** `65537`

### Vulnerability: Weak RSA Modulus
The modulus `n` is 128 bits long. In modern cryptography, a 128-bit modulus is highly insecure and can be factored into its prime components ($p$ and $q$) very quickly.

### Factorization
Using tools like **FactorDB** or the `sympy` library, we factor `n`:
- **p:** `18446744073709551533`
- **q:** `18446744073709551557`

Verification:
$p \times q = 18446744073709551533 \times 18446744073709551557 = 340282366920938460843936948965011886881$ (Success)

## 4. Exploitation: Recovering the Private Key
The private key $d$ is the modular multiplicative inverse of $e$ modulo $\phi(n)$.
$\phi(n) = (p - 1) \times (q - 1)$

**Calculation:**
$\phi(n) = (18446744073709551533 - 1) \times (18446744073709551557 - 1)$
$\phi(n) = 340282366920938460807043459146241379792$

Using the Extended Euclidean Algorithm:
**Private Key (d):** `196442361873243903843228745541797845217`

## 5. Decryption & Flag Retrieval
1. Paste the recovered private key **d** into the ransomware's decryption field.
2. Once the files are decrypted, open the file `student_grades.docx` on the Desktop.

**Flag:** `THM{d0nt_l34k_y0ur_w34k_m0dulu5}`

---
