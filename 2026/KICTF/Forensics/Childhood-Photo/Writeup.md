# Childhood Photo

When I first looked at the challenge, there was only one file:

`gepj.lanif`

The filename immediately felt suspicious. If you reverse it, it becomes `final.jpeg`. That was the first hint that something wasn’t normal.

---

## Initial Inspection

I checked what kind of file it actually was:

```bash
file gepj.lanif
xxd -l 16 gepj.lanif
```

Surprisingly:

* `file` reported it as generic `data`
* The header started with `FF D9`

That’s strange because:

* `FF D8` → JPEG **start** marker
* `FF D9` → JPEG **end** marker

Seeing an end marker at the beginning strongly suggests the file is **byte-reversed**.

![Header Inspection](https://github.com/user-attachments/assets/9a685d9f-b654-4006-805b-3cc5e226bf1f)

---

## Reversing the File

Since the file appeared to be fully reversed, I reversed it byte-by-byte:

```bash
perl -0777 -ne 'print scalar reverse $_' gepj.lanif > final.jpeg
file final.jpeg
```

Now the file was recognized as a valid JPEG image.

This is the image

<img width="576" height="1015" alt="image" src="https://github.com/user-attachments/assets/1ccade42-c8d8-4b28-8380-4443d7c523ed" />


That confirmed the suspicion — the challenge intentionally reversed the entire file to break the signature.

---

## Looking Deeper

In CTF challenges, if something works perfectly, it usually means there’s more hidden underneath.

So I searched for multiple JPEG end markers:

```bash
grep -oba $'\xff\xd9' final.jpeg
wc -c final.jpeg
```

Normally, a JPEG should have only one `FFD9` marker at the very end.

But here, there was an earlier occurrence before the file actually ended.

That means:

* The first JPEG ends earlier
* Extra data is appended after it

---

## Carving the Hidden Payload

I extracted everything after the first `FFD9` marker:

```bash
dd if=final.jpeg of=trailing.bin bs=1 skip=138898 status=none
file trailing.bin
```

![Carved File Type](https://github.com/user-attachments/assets/087f910c-6ca9-459a-a0b4-a30f9739eaa7)

It turned out to be another JPEG.

So I renamed it:

```bash
mv trailing.bin childhood.jpg
```

---

## The Hidden Image

Opening `childhood.jpg` revealed the real content of the challenge.

![Hidden Image](https://github.com/user-attachments/assets/6347a6db-3b9c-489a-b72f-ca061f0b076b)

And there it was — the flag written directly on the image.

---

# Flag

```
KJCTF{r3v3r53d_jp3g_h34d3r_fix3d}
```

---

## Final Thoughts

This challenge used two simple but clever tricks:

* The entire file was byte-reversed to break the JPEG signature.
* A second JPEG image was appended after the first JPEG’s end marker.

No heavy steganography. No encryption. Just understanding file structure and thinking logically.

Sometimes the cleanest tricks are the most satisfying to solve.
