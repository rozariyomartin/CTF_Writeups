# Digital Transition - Writeup

## Challenge Description
We were given a raw signal capture from an HDMI display adapter: [signal.bin](file:///home/martin/Downloads/ctf/2026/oxFun/Hardware/DigitalTransition/signal.bin).
The hint mentioned it contained a single digitized frame from a 640x480 HDMI output.

## Step-1 Analysis
First, I analyzed the file size:
```bash
ls -l signal.bin
# 1680216 bytes
```
A standard 640x480 HDMI signal often includes blanking intervals, resulting in a total frame size of 800x525 pixels.
Calculated size: `800 * 525 * 4 bytes/pixel (RGBA) = 1,680,000 bytes`.
The file size (1,680,216) is slightly larger than the raw data, suggesting a small header or footer.
`1680216 - 1680000 = 216 bytes`.

Using `xxd`, I saw a header and some metadata. The raw pixel data likely starts after this header.

## Step-2
I wrote a Python script to extract the raw pixel data and save it as an image. I assumed a standard 32-bit RGBA format.

### solve.py
```python
import struct
from PIL import Image

def solve():
    with open('signal.bin', 'rb') as f:
        data = f.read()

    # Dimensions for 640x480 with blanking
    width = 800
    height = 525
    bpp = 4 # RGBA
    
    # Calculate offset
    expected_data_size = width * height * bpp
    total_size = len(data)
    header_size = total_size - expected_data_size # 216 bytes

    # Extract pixel data
    pixel_data = data[header_size:]
    
    # Create image
    img = Image.frombuffer('RGBA', (width, height), pixel_data, 'raw', 'RGBA', 0, 1)
    img.save('frame.png')
    print("Saved frame.png")

if __name__ == "__main__":
    solve()
```

## Step-3 Image Analysis
Running the script generated `frame.png`.
Opening this image in a steganalysis tool like **StegSolve** (or simply viewing the color planes) reveals the hidden flag clearly in the noisy active area.

<img width="800" height="525" alt="frame" src="https://github.com/user-attachments/assets/3d8b269f-7870-4d56-8628-261673cb11f8" />

Then I used stegsolve to find the flag.

<img width="807" height="587" alt="image" src="https://github.com/user-attachments/assets/189abb71-750b-4304-9bc3-3bf63c7f7891" />

I found the flag in the alpha plane 4.

## Flag

`0xfun{TMDS_D3C0DED_LIKE_A_PR0}`
