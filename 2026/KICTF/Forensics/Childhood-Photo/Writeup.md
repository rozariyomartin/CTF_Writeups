# Childhood Photo

## Challenge

We were given a single file:

- `gepj.lanif`

The filename is suspicious because it is `final.jpeg` reversed.

---

## Step 1: Identify the file type

Ran:

```bash
file gepj.lanif
xxd -l 16 gepj.lanif
```

Observation:

- `file` reports generic `data`
- Header starts with `FF D9` (JPEG end marker), not `FF D8` (JPEG start marker)

<img width="743" height="168" alt="image" src="https://github.com/user-attachments/assets/9a685d9f-b654-4006-805b-3cc5e226bf1f" />


This strongly suggests the entire file is byte-reversed.

---

## Step 2: Reverse the bytes to recover the original JPEG

```bash
perl -0777 -ne 'print scalar reverse $_' gepj.lanif > final.jpeg
file final.jpeg
```

Result:

- `final.jpeg` is a valid JPEG image.

---

## Step 3: Check for appended/hidden data

Search for JPEG end markers:

```bash
grep -oba $'\xff\xd9' final.jpeg
wc -c final.jpeg
```

Observation:

- There is an earlier `FFD9` before file end, meaning extra bytes are appended after the first image.
- This indicates a second hidden payload.

---

## Step 4: Carve the trailing payload

Extract bytes after the first `FFD9`:

```bash
dd if=final.jpeg of=trailing.bin bs=1 skip=138898 status=none
file trailing.bin
```

Result:

<img width="1470" height="144" alt="image" src="https://github.com/user-attachments/assets/087f910c-6ca9-459a-a0b4-a30f9739eaa7" />


```bash
mv trailing.bin childhood.jpg
```

---

## Step 5: Read the hidden image

Open `childhood.jpg` and inspect visually.

The flag is written on the image:

<img width="564" height="1010" alt="image" src="https://github.com/user-attachments/assets/6347a6db-3b9c-489a-b72f-ca061f0b076b" />

## Flag

`KJCTF{r3v3r53d_jp3g_h34d3r_fix3d}`

---

## Summary

The challenge used two simple tricks:

1. Whole-file byte reversal to break JPEG signature.
2. Appended second JPEG after the first JPEG end marker.

By reversing the original file and carving trailing bytes, the hidden “childhood photo” and flag were recovered.
