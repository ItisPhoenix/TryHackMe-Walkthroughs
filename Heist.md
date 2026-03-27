# TryHackMe: Heist Walkthrough

## Challenge Description
A weakness in the Cipher's Smart Contract could drain all of the ETH in its treasury, thereby breaking the funding to the Phantom Node Botnet and disabling its global malicious operation.

**Target Machine:** 10.49.165.29

---

## 1. Initial Reconnaissance
The first step was to perform an Nmap scan to identify open ports and services on the target.

### Nmap Scan Result:
- **Port 22/tcp:** SSH
- **Port 80/tcp:** HTTP (Hosting a "Blockchain Challenge" app)

The web application on port 80 was built with Vite/React. Inspecting the network traffic and source code revealed several interesting API endpoints:
- `/challenge`: Returns challenge details (RPC URL, Contract Address, Player Wallet).
- `/challenge/source`: Returns the Base64-encoded Solidity source code of the contract.
- `/challenge/solve`: Checks if the challenge is solved and returns the flag.

---

## 2. Analyzing the Smart Contract
Fetching the source code from `/challenge/source` and decoding it revealed the following Solidity contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Challenge {
    address private owner;
    address private initOwner;
    constructor() payable {
        owner = msg.sender;
        initOwner = msg.sender;
    }
    
    function changeOwnership() external {
            owner = msg.sender;
    }
    
    function withdraw() external {
        require(msg.sender == owner, "Not owner!");
        payable(owner).transfer(address(this).balance);
    }
    
    function isSolved() external view returns (bool) {
        return (address(this).balance == 0);
    }
    // ... other helper functions
}
```

### The Vulnerability:
The `changeOwnership()` function is `external` and lacks any access control (e.g., `onlyOwner` modifier). This allows **anyone** to call it and become the owner of the contract. Once ownership is hijacked, the `withdraw()` function can be called to drain all ETH from the contract's treasury.

---

## 3. Exploitation
Using the provided player private key and the Geth RPC endpoint, I performed the following steps using a Python script with `web3.py`:

1.  **Connect to RPC:** Connected to `http://10.49.165.29:8545`.
2.  **Hijack Ownership:** Called the `changeOwnership()` function from the player's wallet.
3.  **Drain Funds:** Called the `withdraw()` function as the new owner to transfer all ETH to the player's wallet.
4.  **Verify:** Checked the contract balance, which became `0`.

---

## 4. Retrieving the Flag
After draining the contract, I made a request to the `/challenge/solve` endpoint to verify the solution.

**Result:**
```json
{"flag":"THM{web3_h31st_d0ne}"}
```

**Flag:** `THM{web3_h31st_d0ne}`

---

