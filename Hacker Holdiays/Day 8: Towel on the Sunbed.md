## Walkthrough: TryHackMe Room: "Towel on the Sunbed"

A comprehensive, step-by-step writeup analyzing and exploiting the **Ponzi — Wellness Rewards** web challenge featured in TryHackMe's Hacker Holidays event. This challenge showcases a classic business logic flaw running on a Node.js/Express stack: a **Race Condition (Time-of-Check to Time-of-Use / TOCTOU)** within a daily rewards claim pipeline.

---

## 📊 Room Summary

| Field | Value |
|-------|-------|
| **Room Name** | Ponzi — Wellness Rewards |
| **Category** | Web Exploitation / Business Logic |
| **Difficulty** | Medium |
| **Target Endpoint** | `http://10.48.162.95:3000` |

---

# 🗺️ Phase 1: Vulnerability Identification & Threat Analysis

## 1. Functional Assessment

Navigating to the application interface reveals a hospitality-themed crypto rewards portal. Registering a guest account exposes the core engine component: a **Claim Daily Reward** button.

Clicking the button grants an initial balance increment of **50 points** and triggers a client-side JavaScript block that locks down consecutive clicks by displaying a **24-hour countdown timer**.

The objective is to elevate the account to **Whale Status** in order to unlock the privileged **Whale Vault**.

---

## 2. Sifting Through Cues

The room description provides several hints pointing toward a concurrency issue:

> *"He came back to find the sunbed had been 'claimed' three times over while he wasn't looking."*

> *"Somewhere between his request and the server's clock, there's a gap wide enough to walk a whale through."*

### Vulnerability Theory

When the reward endpoint receives a request, the backend performs the following sequence:

1. Check whether the last reward timestamp is older than 24 hours (**Time-of-Check**).
2. If valid, calculate and award the reward.
3. Update the user's balance.
4. Save the new timestamp (**Time-of-Use**).

If multiple requests reach the server at nearly the exact same moment, each request may complete the validation step **before** any request updates the timestamp.

As a result, multiple requests can all pass validation and receive rewards.

```
Thread A
    │
    ├── Check timestamp ✔
    │
Thread B
    │
    ├── Check timestamp ✔
    │
Thread C
    │
    ├── Check timestamp ✔
    │
──────────── Race Window ────────────
    │
Thread A → Update timestamp
Thread B → Update timestamp
Thread C → Update timestamp
```

This is a classic **TOCTOU (Time-of-Check to Time-of-Use)** race condition.

---

# 🚀 Phase 2: Exploitation

## 1. Endpoint Interception

The browser prevents repeated clicks using client-side JavaScript.

Instead of interacting with the UI, intercept the underlying HTTP request.

### Steps

1. Register a fresh account.
2. Enable **Burp Suite → Proxy → Intercept**.
3. Click **Claim Daily Reward**.
4. Capture the request.

Example:

```http
POST /claim HTTP/1.1
Host: 10.48.162.95:3000
Cookie: connect.sid=s%3A...
```

---

## 2. Preparing Parallel Requests

Sending requests sequentially will not work because the server updates the timestamp after the first successful request.

Instead, the goal is to make many requests reach the server simultaneously.

### Steps

1. Send the intercepted request to **Repeater** (`Ctrl + R`).
2. Drop the original intercepted request from Proxy so the reward remains unclaimed.
3. Duplicate the request approximately **20 times**.
4. Place every request into a **Repeater tab group**.

---

## 3. Using Last-Byte Synchronization

Burp Suite provides a synchronization mechanism called **Last-Byte Sync**.

### Steps

1. Click the dropdown beside **Send**.
2. Select:

```
Send group in parallel (last-byte sync)
```

3. Click **Send Group**.

---

### How Last-Byte Sync Works

```
Attack Machine
      │
      ├────────────── Send Headers ─────────────► Target
      │
      ├──── Hold Final Byte ───────────────────► Target
      │
      └──── Release Final Byte (All Together) ─► Target
```

Burp sends almost the entire request for every connection while withholding the final byte.

Once every request is waiting on the server, Burp releases the last byte simultaneously.

This causes all requests to begin processing at nearly the same instant, dramatically increasing the probability of winning the race condition.

---

# 🏆 Phase 3: Validation & Flag Retrieval

## 1. Reviewing Responses

Some requests will inevitably arrive after the timestamp has already been updated.

Those requests receive:

```http
429 Too Many Requests
```

However, several requests should complete successfully.

Example response:

```json
{
  "message": "Staking reward claimed successfully.",
  "reward": 50,
  "newBalance": 650,
  "tier": "whale",
  "priceSnapshot": 4.2
}
```

The concurrent requests successfully bypass the intended one-claim-per-day restriction, increasing the balance to **650** and promoting the account to the **Whale** tier.

---

## 2. Retrieving the Flag

1. Return to the browser.
2. Perform a hard refresh (`Ctrl + F5`).
3. Observe the upgraded balance.
4. Open the **Whale Vault**.
5. Retrieve the flag.

**Flag**

```text
THM{......................}
```

---

# 🔬 Root Cause Analysis

The vulnerability exists because the application separates **validation** from **state modification**.

```
Receive Request
       │
       ▼
Check Timestamp
       │
       ▼
Reward User
       │
       ▼
Update Timestamp
```

Since multiple requests execute concurrently, each one validates the old timestamp before any update has been committed.

The application therefore violates atomicity.

---

# 🛡️ Remediation

## 1. Atomic Database Operations

Perform the validation and update in a **single atomic database transaction**.

Examples include:

- Conditional updates
- Row locking
- Transactions
- Atomic increment operations

---

## 2. Session-Level Mutex

Introduce a per-user lock.

```
Incoming Request
        │
Acquire User Lock
        │
Already Locked?
     │        │
    Yes      No
     │        │
 Reject     Process
     │        │
 Release Lock
```

Only one `/claim` request should execute at any given time for a specific user.

---

## 3. Database Row Locking

Use:

- `SELECT ... FOR UPDATE`
- Optimistic locking
- Serializable transactions

to prevent concurrent modifications.

---

## 4. Idempotency Tokens

Assign a unique token to each reward claim.

If the same token is submitted more than once, reject duplicate requests before processing.

---

# 📚 Key Takeaways

- Client-side protections are never security controls.
- Business logic flaws can be as severe as traditional injection vulnerabilities.
- Race conditions often occur when validation and updates are separated.
- Parallel request synchronization can expose hidden concurrency bugs.
- Critical state-changing operations should always be atomic.

---

# 🏁 Conclusion

The **Ponzi — Wellness Rewards** challenge demonstrates a textbook **TOCTOU race condition** within a daily reward system.

Although the web interface prevents repeated clicks through client-side JavaScript, the backend fails to synchronize concurrent requests properly. By leveraging Burp Suite's **Last-Byte Synchronization**, multiple reward claims are processed before the timestamp is updated, allowing an attacker to bypass the intended daily restriction, reach **Whale Status**, and access the protected **Whale Vault**.

This challenge highlights why security-critical business logic should rely on **atomic operations**, **locking mechanisms**, and **proper transaction handling** rather than sequential application-level validation.
