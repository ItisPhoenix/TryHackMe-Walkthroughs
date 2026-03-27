# PassCode - TryHackMe Walkthrough

**Target IP:** 10.49.174.244
**Challenge:** Web3/Blockchain

## 1. Enumeration

### Nmap Scan
An initial Nmap scan revealed the following open ports:
- **Port 22:** SSH
- **Port 80:** HTTP (Vite/React frontend)
- **Port 8545:** Ethereum JSON-RPC endpoint

### Web Analysis
The website is a frontend for a blockchain challenge. Checking the `/challenge` and `/challenge/source` endpoints provided critical information:
- **Contract Address:** `0xf22cB0Ca047e88AC996c17683Cee290518093574`
- **Wallet Address:** `0x3524c71729b3d28a843D96158C9E1107b52a030A`
- **Source Code:** Base64 encoded Solidity contract.

## 2. Vulnerability Analysis

### Smart Contract Logic
The contract `Challenge` stores a `secret` (the flag) and a `code` (passcode) as `private` variables. In Solidity, `private` visibility only prevents other contracts from reading the variable; however, the data is still stored on the public blockchain and can be read by anyone querying the contract's storage slots.

### Storage Layout
- **Slot 0:** `secret` (string)
- **Slot 1:** `unlock_flag` (bool)
- **Slot 2:** `code` (uint256)
- **Slot 3:** `hint_text` (string)

## 3. Exploitation

### Reading Private Storage via RPC
Using the `eth_getStorageAt` method, we can directly query the storage slots of the contract.

#### Retrieving the Passcode (Slot 2):
Querying slot 2 returned `0x...14d`, which is **333** in decimal.

#### Retrieving the Flag (Slot 0):
Querying slot 0 returned the hex-encoded flag.
Hex: `54484d7b776562335f6834636b316e675f636f64657d`
Decoded: `THM{web3_h4ck1ng_code}`

## 4. Conclusion
The challenge demonstrates that "private" variables in smart contracts are not truly secret. Sensitive data should never be stored directly in contract state variables if confidentiality is required.

**Flag:** `THM{web3_h4ck1ng_code}`
**Passcode:** `333`
