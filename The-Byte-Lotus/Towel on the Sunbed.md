# TryHackMe Writeup: Towel on the Sunbed

> **Room:** Towel on the Sunbed  
> **Category:** Web Exploitation / Business Logic Abuse  
> **Difficulty:** Medium

---

## 📖 Room Overview

The resort's wellness portal hosts a crypto rewards application called **Ponzi**. Users receive **50 PONZI** every 24 hours by claiming a daily staking reward.

The objective is to discover a flaw in the reward system, abuse it to accumulate enough points to become a **Whale**, and retrieve the flag from the **Whale Vault**.

---

# 🛠️ Enumeration

## Nmap Scan

```bash
nmap -sC -sV -O <TARGET_IP>
```

### Result

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu
3000/tcp open  http    Node.js Express framework

http-title:
Ponzi Portfolio — Login
```

### Observations

- SSH was available but not relevant.
- A **Node.js Express** web application was running on **port 3000**.
- The application redirected users to:

```
/auth/login
```

This suggested that authentication would be required before interacting with the application.

---

# 🔍 Application Exploration


After registering a new account and logging in, the dashboard displayed:

- Portfolio Balance
- Market Prices
- Daily Reward
- Whale Vault Progress

 
The application provided:

- User Registration
- Login
- Portfolio Dashboard
- Daily Staking Reward
- Whale Vault


The daily reward granted:

```
+50 PONZI
```

After claiming the reward once, the application displayed a **24-hour cooldown**.

At first glance, this appeared to prevent multiple claims.

---

# 💡 Identifying the Vulnerability

The room description contains several important hints:

> claimed three times

> once every 24 hours

> somewhere between his request and the server's clock

These hints strongly suggest a **Race Condition** (TOCTOU - Time Of Check To Time Of Use).

The idea is simple:

If several reward requests reach the server simultaneously, they may all pass the validation **before** the cooldown timestamp is updated.

---

# 🕵️ Intercepting the Request

Using **Burp Suite**, intercept the reward request.

```
POST /claim HTTP/1.1
Host: TARGET:3000
Cookie: connect.sid=...
```

The request contained no body.

The reward logic relied entirely on the authenticated session.

---

# ⚡ Exploiting the Race Condition

Instead of sending a single request, duplicate the request several times inside **Burp Repeater**.

Create **4 identical tabs**, then:

```
Select all tabs
        ↓
Group them
        ↓
Send Group (parallel)
```

Burp transmits every request simultaneously.

Since the application checked the cooldown before updating it, every request was accepted.

---

# 📥 Server Response

One of the successful responses returned:

```json
{
    "message": "Staking reward claimed successfully.",
    "reward": 50,
    "newBalance": 200,
    "tier": "Whale",
    "priceSnapshot": 4.2
}
```

The account balance immediately increased to:

```
200 PONZI
```

Although only **150 PONZI** was required, sending four parallel requests resulted in a total balance of **200 PONZI**.

<img width="1539" height="636" alt="8burpparralel" src="https://github.com/user-attachments/assets/46145d88-04df-4eba-b517-4b22d8c9ab57" />

---

# 🐋 Whale Vault

Once the balance exceeded **150 PONZI**, the dashboard unlocked the **Whale Vault**.

Click:

```
Open Vault
```
<img width="746" height="844" alt="8reward" src="https://github.com/user-attachments/assets/16624d6c-9390-4578-90a8-f3aa2bd8a837" />


The application revealed the flag.

---


# 🚩 Flag

```text
THM{t0w3l_0n_th3_*******_d0ubl3_sp3nt}
```

---

# 📚 Root Cause Analysis

The reward process likely followed this sequence:

```text
Receive Request
        │
        ▼
Check cooldown
        │
        ▼
Award reward
        │
        ▼
Update cooldown
```

When multiple requests arrived simultaneously:

```text
Request 1 ──► Check ✓
Request 2 ──► Check ✓
Request 3 ──► Check ✓
Request 4 ──► Check ✓
                 │
                 ▼
All requests award 50 PONZI
                 │
                 ▼
Cooldown updated afterwards
```

Each request observed that the reward had not yet been claimed.

As a result, every request awarded **50 PONZI** before the cooldown timestamp was finally written.

This is a classic **Race Condition (TOCTOU)** vulnerability.

---

# 💻 Tools Used

- Nmap
- Burp Suite Repeater
- Firefox

---

# 📖 What I Learned

- Business Logic vulnerabilities do not rely on traditional injection attacks.
- Race Conditions occur when multiple requests execute before shared application state is updated.
- Parallel request testing is an effective technique for identifying timing issues.
- Burp Repeater's **Send Group (parallel)** feature is useful for testing concurrent request handling.

---

# ✅ Attack Flow

```text
Nmap Scan
     │
     ▼
Discover Web Application
     │
     ▼
Register New Account
     │
     ▼
Login
     │
     ▼
Claim Daily Reward
     │
     ▼
Intercept Request with Burp
     │
     ▼
Duplicate Request ×4
     │
     ▼
Send Group (parallel)
     │
     ▼
Receive Multiple Rewards
     │
     ▼
Reach Whale Tier
     │
     ▼
Open Whale Vault
     │
     ▼
Capture Flag 🚩
```

---

## 🎯 Key Takeaway

This room demonstrates how a seemingly secure cooldown mechanism can fail when the application does not handle concurrent requests safely. By exploiting a race condition with Burp Suite's parallel request feature, it was possible to claim the daily reward multiple times before the server updated the cooldown, resulting in enough PONZI to unlock the Whale Vault.

---
