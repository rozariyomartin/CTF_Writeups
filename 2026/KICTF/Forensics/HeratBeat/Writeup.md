# Heart Beat - Forensics CTF Writeup

## Challenge Description
*An anonymous source leaked this strange animation. It plays. It loops. It looks normal.*

*The first thing you see is never the last thing you find. Every frame has a heartbeat, measured in 1/100th of a second. 100 heartbeats in total — but only certain numbers have always stood apart from the rest in mathematics. What do those special moments whisper when you listen closely?*

## 🔍 Initial Thoughts & Clues
When I first read the description, a few things immediately stood out like puzzle pieces waiting to be put together:
1.  **"100 heartbeats in total"** -> The animation (GIF) has exactly 100 frames.
2.  **"measured in 1/100th of a second"** -> The delay time between frames in a GIF is stored in hundredths of a second. This "heartbeat" is the *frame delay*.
3.  **"only certain numbers have always stood apart from the rest in mathematics"** -> A classic reference to **Prime Numbers**. 
4.  **"What do those special moments whisper"** -> If we take the frame delays at prime-numbered frames, they probably represent ASCII characters (since standard text characters fall between 32 and 126). 

So the plan became clear: Find the delay for each frame, grab the ones that are prime-numbered, and covert those delay values into text!

## The Process

Instead of using heavy image-processing libraries which can sometimes be slow or get confused by corrupted chunks, I decided to parse the raw structure of the GIF file directly using Python. 

In the GIF format, the frame delays are stored inside the `Graphic Control Extension` block, which always starts with the bytes `21 F9 04`. The actual delay time is stored right after that. 

Here is the quick script I put together to read those raw bytes and extract the hidden whispers:

```python
import struct

# Helper function to check if a number is prime
def is_prime(n):
    if n < 2: return False
    for p in range(2, int(n**0.5)+1):
        if n % p == 0: return False
    return True

# Read the raw GIF data
with open('animation.gif', 'rb') as f:
    data = f.read()

delays = []
idx = 0

# Scan through the raw bytes looking for the Graphic Control Extension block (0x21 0xF9 0x04)
while True:
    idx = data.find(b'\x21\xf9\x04', idx)
    if idx == -1:
        break
    
    # The delay is a 2-byte value stored 4 bytes into the block (little-endian format)
    delay = struct.unpack('<H', data[idx+4:idx+6])[0]
    delays.append(delay)
    
    # Move past this block to find the next one
    idx += 8

# The challenge implies the "first" frame, so we need to decide if we start counting at 0 or 1.
# Usually, in human terms, the first frame is frame "1", so 1-indexed prime checking makes the most sense.
flag1 = ''.join(chr(d) for i, d in enumerate(delays) if is_prime(i + 1))

print(f"Extracted Flag: {flag1}")
```

## The Payload Delivery

Running the script immediately pulled the delay timings, checked if their frame index was prime, and translated the delay values into ASCII text. 

```bash
$ python3 solve.py
Extracted Flag: KICTF{th15_15_pr1m3_t1m3}
```

The whispered message from those special mathematical moments turned out to be the exact flag. 

**Flag:** `KICTF{th15_15_pr1m3_t1m3}` 🎉
