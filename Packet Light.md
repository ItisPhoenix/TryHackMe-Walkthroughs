# Walkthrough: TryHackMe Room: "Packet Light" 

## Challenge Description

> Analyze the provided network capture for a covert communication channel.
>
> Identify where the exfiltrated data is being hidden, reassemble it, decode the recovered data, and submit the flag.

---

# Objective

The goal of this challenge is to identify how sensitive information is being covertly transmitted over the network, reconstruct the hidden data, decode it, and recover the final flag.

---

# Tools Used

- Wireshark
- tshark (optional)
- Python
- CyberChef (optional)

---

# Step 1 — Inspect the Network Capture

Open the provided PCAP file in Wireshark.

The capture contains several HTTP requests between a client and a web server.

Rather than looking only at URLs or POST bodies, inspect the HTTP headers.

Navigate to:

```
Hypertext Transfer Protocol
```

One header immediately stands out:

```
Cookie:
```

Among the normal cookies is a custom cookie:

```
hotel_sess_state
```

Example:

```
Cookie:
    PHPSESSID=...
    hotel_sess_state=Jw==
```

This cookie appears in multiple requests.

Since cookies are rarely used to transmit changing encoded values, it is a strong indicator of a covert communication channel.

---

# Step 2 — Follow the HTTP Stream

Follow the TCP/HTTP stream.

During the session the client downloads a Python script:

```
GET /temp/updates.py
```

This file contains the malware responsible for the communication.

---

# Step 3 — Analyze the Malware

Reviewing the Python source reveals the exfiltration routine.

The malware performs the following operations:

1. Capture a keystroke
2. XOR the character using a hardcoded key
3. Base64 encode the XOR result
4. Send the encoded value inside the HTTP Cookie header

Simplified pseudocode:

```python
key = "H0t3lSt@ff0NlyK3epS3cr3t!"

encrypted = xor(character, key)
encoded = base64.b64encode(encrypted)

Cookie:
hotel_sess_state=<encoded>
```

The hardcoded XOR key is:

```
H0t3lSt@ff0NlyK3epS3cr3t!
```

This immediately explains why every HTTP request contains a different value for the cookie.

---

# Step 4 — Extract the Hidden Data

Every HTTP request contains one encoded character.

Extract every occurrence of

```
hotel_sess_state=
```

Example values:

```
Jw==
Mg==
Kw==
...
```

These values must be collected **in chronological order**, since each represents one character of the original message.

This can be done using:

- Wireshark
- tshark
- Python

---

# Step 5 — Decode Each Cookie

Each cookie value undergoes two decoding stages.

## Stage 1 — Base64 Decode

Example:

```
Jw==
```

↓

```
'
```

This produces the XOR-encrypted byte.

---

## Stage 2 — XOR Decode

Using the recovered key:

```
H0t3lSt@ff0NlyK3epS3cr3t!
```

XOR each byte with the corresponding key byte.

Pseudo-code:

```python
decoded_char = encrypted_byte ^ key[index % len(key)]
```

Perform this operation for every cookie.

---

# Step 6 — Reassemble the Message

After decoding every cookie and concatenating the recovered characters in order, the hidden message becomes:

```
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

---

# Flag

```
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

---

# Technical Analysis

The malware does **not** send data inside:

- HTTP POST body
- URL parameters
- DNS queries
- HTTP response data

Instead, it abuses the **HTTP Cookie header** as its covert communication channel.

The complete exfiltration pipeline is:

```
Keystroke
      │
      ▼
 XOR Encryption
      │
      ▼
 Base64 Encoding
      │
      ▼
 HTTP Cookie Header
      │
      ▼
 Network Traffic
```

This technique is effective because:

- Cookies are routinely transmitted with HTTP requests.
- Security tools often prioritize payload inspection over header analysis.
- The encoded values resemble legitimate session data.
- Only a single encoded character is transmitted per request, minimizing suspicion.

---

# Detection Opportunities

A defender could identify this behavior by monitoring for:

- Rapidly changing cookie values
- High-entropy cookies
- Unrecognized custom cookie names
- Consistent outbound HTTP requests carrying small encoded payloads
- Downloads of suspicious Python scripts followed by repeated HTTP beaconing

---

# Conclusion

The covert communication channel was implemented using a custom HTTP cookie named `hotel_sess_state`. The malware XOR-encrypted each captured keystroke with a hardcoded key, Base64-encoded the result, and transmitted it as a cookie value. By extracting the cookie values from the PCAP, Base64-decoding them, XORing with the recovered key, and concatenating the resulting characters in sequence, the hidden message was reconstructed successfully.

Recovered Flag:

```
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```