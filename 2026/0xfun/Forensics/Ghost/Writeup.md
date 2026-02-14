# Ghost - Writeup

<img width="521" height="482" alt="image" src="https://github.com/user-attachments/assets/8c2d3009-f43c-4a00-a53f-8c45c4ad6d7e" />


 _“The truth is in the static.”_  
A layered forensic challenge combining PCAP analysis, SSTV, and steganography — with heavy misdirection.


## 1) Starting Artifact — Intercepted Email

The challenge provides an email file (`help.eml`) containing suspicious headers and RF-style metadata.

Key narrative clues:

- “only a network capture remaining”
    
- “The truth is in the static”
    
- Numerous radio-communication parameters
    
- ProtonMail / Tor context (lore)
    

This strongly suggests **signal-based hidden data inside the capture**, not in the email body itself.

---

## 2) Follow the Lead — PCAP File

A link in the headers leads to a download of:

`capture.pcap`

Open in Wireshark and extract transferred files:

`File → Export Objects → HTTP`

Recovered artifacts include:

`status avatar.jpeg profile.jpg voila.png 1.wav 2.mp3 lost.wav hint.wav idk.png ...`

---

## 3) “Truth is in the static” → SSTV Decode

The file named **`status`** contains an SSTV transmission.

Decode using tools such as:

- QSSTV (Linux)
    
- RX-SSTV
    
- MMSSTV (via Wine)
    
- Online SSTV decoder
    

The decoded image reveals leetspeak text:

`1n73rc3p7_c0nf1rm3d`

This is the **passphrase** for the next stage.

---

## 4) Image Steganography — avatar.jpeg

Suspicious images are tested with steganography tools.

Use **steghide** with the SSTV passphrase:

`steghide extract -sf avatar.jpeg`

Passphrase:

`1n73rc3p7_c0nf1rm3d`

Extraction produces:

`key.txt`

Contents:

`l4y3r_pr0t3c710n_k3y`

---

## 5) Identify the Flag

Convert leetspeak:

`l4y3r_pr0t3c710n_k3y → layer_protection_key`

Given the challenge flag format:

`0xfun{...}`

The recovered key is already:

✔ Meaningful  
✔ Thematically correct  
✔ Directly obtained from hidden data  
✔ Flag-shaped

---

## 🏁 Final Flag

`0xfun{l4y3r_pr0t3c710n_k3y}`
