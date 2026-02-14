# Nothing Expected - Writeup

<img width="514" height="450" alt="image" src="https://github.com/user-attachments/assets/e9a929c1-2ee1-4d21-b257-c206a6a842eb" />

The challenge name **“NothingExpected”** suggests the file may appear normal but hides something in an unexpected place.

## Step 1 — Initial Analysis

Check file type:

<img width="575" height="107" alt="image" src="https://github.com/user-attachments/assets/871cfb3c-df8d-4eb0-8679-14d4c77ef588" />


The image opens normally — nothing suspicious visually.

---

## Step 2 — Steganography Scan

Run `zsteg`, a common PNG stego tool:

<img width="687" height="365" alt="image" src="https://github.com/user-attachments/assets/894651c5-c794-4965-80ce-e9865a22c138" />


Output reveals:

`meta application/vnd.excalidraw+json.. file: JSON text data`

👉 This is the key clue.

The PNG contains embedded metadata of type:

`application/vnd.excalidraw+json`

This MIME type belongs to **Excalidraw**, a diagram-drawing tool.

---

## Step 3 — Understanding Excalidraw PNG Exports

Excalidraw PNG exports embed the original drawing scene inside PNG metadata (usually iTXt chunks).  
This allows the image to be reopened and edited later.

So the PNG is not just an image — it contains a hidden drawing.

---

##  Step 4 — Failed Extraction Attempts

Typical methods were attempted:

### Binwalk

`binwalk -eM file.png`

<img width="813" height="428" alt="image" src="https://github.com/user-attachments/assets/a6d70f7b-8636-418f-8691-b2166709d498" />


Only partial zlib data detected — nothing usable extracted.

---

### Strings Extraction

`strings file.png > dump.txt`

Extracted fragments were truncated and not valid JSON.

---

### Exiftool

`exiftool file.png`

<img width="1900" height="945" alt="image" src="https://github.com/user-attachments/assets/9cc48a63-46b0-427e-ab41-7b2d7bd48d83" />



---

## 🏆 Step 5 — Intended Solution

Since the data is an embedded Excalidraw scene, the simplest method is:

### 👉 Open the PNG directly in Excalidraw

1. Go to:
    

`https://excalidraw.com`

2. Menu → **Load → From file**
    
3. Select `file.png`
    

---

## 🔥 Step 6 — Revealing the Hidden Content

After loading, the hidden drawing appears.

<img width="1310" height="399" alt="image" src="https://github.com/user-attachments/assets/6538cd2d-2bf3-4b86-a8ed-6d6326416478" />


---

## 🚩 Flag

`0xfun{th3_sw0rd_0f_k1ng_4rthur}`

---
