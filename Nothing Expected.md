![[Pasted image 20260213112142.png]]

The challenge name **“NothingExpected”** suggests the file may appear normal but hides something in an unexpected place.

## Step 1 — Initial Analysis

Check file type:

![[Pasted image 20260213112314.png]]

The image opens normally — nothing suspicious visually.

---

## Step 2 — Steganography Scan

Run `zsteg`, a common PNG stego tool:

![[Pasted image 20260213112347.png]]

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

![[Pasted image 20260213112435.png]]

Only partial zlib data detected — nothing usable extracted.

---

### Strings Extraction

`strings file.png > dump.txt`

Extracted fragments were truncated and not valid JSON.

---

### Exiftool

`exiftool file.png`

![[Pasted image 20260213112539.png]]


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

![[Pasted image 20260213112759.png]]

---

## 🚩 Flag

`0xfun{th3_sw0rd_0f_k1ng_4rthur}`

---