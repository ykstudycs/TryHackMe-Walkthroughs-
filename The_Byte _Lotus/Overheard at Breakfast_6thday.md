# 🥐 TryHackMe Writeup: Overheard at Breakfast

> **Category:** OSINT (Open Source Intelligence)
> **Difficulty:** Easy

---

# 📖 Room Overview

In this challenge, we are given a screenshot of a private conversation that was supposedly overheard during breakfast at the Byte Lotus resort.

Our objective is to carefully analyze the conversation, extract identifying information, locate a hidden online profile, and recover the final flag.

---

# 🎯 Objective

* Analyze the provided conversation
* Extract identifying clues
* Locate the hidden online account
* Recover the hidden flag

---

# 📂 Step 1 – Download the Evidence

Download the file provided in the task.

```
overheard-at-breakfast-1784259780309.zip
```

Extract the archive:

```bash
unzip overheard-at-breakfast-1784259780309.zip
```

Output:

```text
Archive:  overheard-at-breakfast-1784259780309.zip
inflating: conversation.png
```

The archive contains a single image:

```
conversation.png
```
<img width="1175" height="781" alt="conversation" src="https://github.com/user-attachments/assets/91a01d88-d8a7-4c83-9bf8-ebff86789a21" />


---

# 🖼️ Step 2 – Read the Conversation Carefully

The screenshot contains a conversation between two users:

* **Ponzi - Influencer**
* **Lambo!**

The most important part of the conversation is this section:

> "I don't really use much social media...
>
> Though I'm still out there, I used to use this free tool that let me upload my profile and link other media accounts.
>
> Started with a **G** if I remember correctly.
>
> But if anything this is my best way of communication:lambobytelotushotel@gmail.com
>


---

# 🔍 Step 3 – Extract the Clues

Rather than skimming the conversation, read every line carefully.

The following pieces of information stand out.

| Clue                                                                      | Why it matters               |
| ------------------------------------------------------------------------- | ---------------------------- |
| **Lambo!**                                                                | Username/Nickname            |
| **Ponzi - Influencer**                                                    | Conversation participant     |
| **Byte Lotus**                                                            | Resort name (context)        |
| **[lambobytelotushotel@gmail.com](mailto:lambobytelotushotel@gmail.com)** | Strong identifier            |
| **"Started with a G"**                                                    | Hint about an online service |

The room hint also encourages us to **read the conversation carefully**, not just skim through it.

---

# 🧠 Step 4 – Identify the Service

The conversation describes a service that:

* Starts with the letter **G**
* Lets users upload a profile
* Links other social media accounts
* Is associated with an email address

Several services may come to mind:

* Google
* GitHub
* GitLab
* Goodreads
* **Gravatar**

Among these, **Gravatar** perfectly matches the description.

Gravatar allows users to create a global profile associated with an email address and link other social accounts.

---

# 🌐 Step 5 – Search the Email

Open Gravatar and use its email lookup feature.
<img width="1057" height="355" alt="emailchecker" src="https://github.com/user-attachments/assets/05c8206d-3938-4033-8403-2294cdbd2a90" />




Search for:

```
lambobytelotushotel@gmail.com
```

The lookup reveals:

```text
Profile URL:
https://gravatar.com/d43faafe9d7f056793bd037b8d6e321acad985c222d83775b10d6539e301e931

Email Hash:
d43faafe9d7f056793bd037b8d6e321acad985c222d83775b10d6539e301e931

Avatar URL:
/avatar/d43faafe9d7f056793bd037b8d6e321acad985c222d83775b10d6539e301e931
```

<img width="980" height="882" alt="gravatar" src="https://github.com/user-attachments/assets/4b971b1b-a0f0-454f-8ddc-726d3093facb" />


---

# 🔎 Step 6 – Visit the Profile

Open the discovered profile URL.

The profile contains the following message:

> Funny thing about email hashes, they follow you places you didn't expect.
>
> Glad you found the right corner of the internet!
>
> Here is your prize:
>
> ```
> VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9
> ```

This looks like a **Base64-encoded** string.

<img width="860" height="607" alt="lambofinal" src="https://github.com/user-attachments/assets/a20b111d-f168-41b4-9f2e-4a0a40bb7c0c" />


---

# 🔐 Step 7 – Decode the String

Decode the Base64 value.

```text
VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM***
```

Using any Base64 decoder:

```bash
echo "VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50M****" | base64 -d
```

Output:

```text
THM{S3creT_Pr**************_Ident1fi3d}
```

---

# 🚩 Flag

```text
THM{S3creT_Pr**************_Ident1fi3d}
```

---

# 💡 Key Takeaways

* Always read every line carefully during OSINT challenges.
* Email addresses are powerful identifiers that can reveal linked online profiles.
* Gravatar profiles are tied to email addresses and may expose additional information.
* Small conversational hints (such as *"started with a G"*) can completely change the direction of an investigation.
* Encoded strings encountered during investigations are often Base64 and should always be tested.

---

# 🛠️ Skills Practiced

* OSINT Investigation
* Evidence Analysis
* Information Extraction
* Email Enumeration
* Gravatar Profile Discovery
* Base64 Decoding
* Attention to Detail

---

## 🏁 Conclusion

This room demonstrates how seemingly harmless pieces of information shared in casual conversations can be combined to uncover someone's online identity. A single email address, paired with a subtle hint about a profile-linking service, was enough to locate a forgotten Gravatar profile and retrieve the hidden flag.

The biggest lesson from this challenge is that successful OSINT investigations rarely rely on complex techniques—they rely on careful observation, logical reasoning, and making connections between small pieces of publicly available information.
