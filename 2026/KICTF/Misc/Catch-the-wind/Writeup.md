# Catch the Wind

**Category:** Forensics
**Flag:** `KICTF{0FF_7h3_r4d4r_n0w}`

*The attacker claims they bypassed perimeter security and left a “parting gift” in an outbound broadcast. They challenge us to reconstruct the stream and see the big picture.*


When I first read the challenge description, two phrases immediately stood out:

**“Reconstruct the stream.”**
**“See the big picture.”**

That’s not random wording. In forensics challenges, those are instructions disguised as flavor text.

So I opened `handout.pcap` in Wireshark to see what we were dealing with.


## First Impressions

At a glance, the capture looks boring. Standard network noise — HTTP requests, NTP traffic, nothing screaming “malware.”

But scrolling through the packet payloads, something unusual pops up.

A huge portion of the traffic isn’t web data at all.

It’s GPS telemetry.

More specifically, repeated **`$GPGGA` NMEA sentences**.

That’s strange. Why would GPS coordinates be flying around in this network capture?

And then it clicked — *“see the big picture.”*

GPS coordinates aren’t meant to be read as text.

They’re meant to be plotted.


## Extracting the Data

To make life easier, I dumped all printable strings from the PCAP:

<img width="551" height="81" alt="image" src="https://github.com/user-attachments/assets/eb997423-b49c-424a-ae48-cdfa600e7eb8" />

Opening `capture.txt`, the `$GPGGA` lines stand out immediately.

If you try to plot all of them directly, though, you get absolute chaos. The graph becomes a tangled mess with lines constantly snapping back to a weird point on the far left.

That’s not random noise.

That’s intentional sabotage.

---

## The Attacker’s “Parting Gift”

Looking closer at the GPS sentences, I noticed two patterns.

<img width="1872" height="1005" alt="image" src="https://github.com/user-attachments/assets/40996c57-c5fd-4323-86d6-7892a6fc8725" />

### Legitimate entries looked like this:

```
$GPGGA,143000.000,3218.7522,S,04131.1573,W,1,08,0.9,120.5,M,0.0,M,,*6F
```

They end with a proper hexadecimal checksum like `*6F`.

### Suspicious entries looked like this:

```
$GPGGA,143002.000,1823.4400,S,17330.1200,W,1,06,1.2,5.0,M,0.0,M,,*XX
```

That `*XX` checksum isn’t valid.

And the coordinates often pointed to the same strange location.

That’s the “parting gift.”

The attacker injected spoofed GPS data to ruin the visualization.

If you plot everything blindly, you’ll never see the message.

So the fix is simple:

Filter out any `$GPGGA` line containing `*XX`.

---

## Converting the Coordinates

NMEA coordinates use `DDMM.MMMM` format (Degrees + Decimal Minutes).

To plot them correctly, you need to convert them to standard Decimal Degrees:

```
Decimal Degrees = Degrees + (Minutes / 60)
```

And don’t forget:

* `S` → negative latitude
* `W` → negative longitude

Once converted, they can be plotted on a normal Cartesian plane.

```
import matplotlib.pyplot as plt

# 1. Read the extracted strings from the PCAP
with open('capture.txt', 'r') as f:
    lines = f.readlines()

x_coords = []
y_coords = []

# 2. Parse the coordinates
for line in lines:
    # Filter out the attacker's injected spoof data (*XX)
    if '$GPGGA' in line and '*XX' not in line:
        parts = line.split(',')
        
        # Ensure the line has the required latitude/longitude fields
        if len(parts) >= 6 and parts[2] and parts[4]:
            lat_raw = float(parts[2])
            lat_dir = parts[3]
            lon_raw = float(parts[4])
            lon_dir = parts[5]
            
            # Convert NMEA DDMM.MMMM to Decimal Degrees
            lat = (int(lat_raw / 100) + (lat_raw % 100) / 60)
            if lat_dir == 'S': lat = -lat
                
            lon = (int(lon_raw / 100) + (lon_raw % 100) / 60)
            if lon_dir == 'W': lon = -lon
                
            x_coords.append(lon)
            y_coords.append(lat)

# 3. Draw the "Big Picture"
plt.figure(figsize=(15, 5))
# Plot the points with lines connecting them sequentially
plt.plot(x_coords, y_coords, marker='o', linestyle='-', linewidth=2)
plt.title("CTF GPS Path (Spoofed Points Removed)")
plt.xlabel("Longitude")
plt.ylabel("Latitude")

# Save the image (required for headless/WSL environments)
plt.savefig('clean_flag.png')
print("Plot saved as clean_flag.png!")
```

## The Moment It Clicked

After filtering out the spoofed lines and plotting only the legitimate coordinates, the chaos disappeared.

Instead of a messy scribble, the points formed clean, connected lines.

<img width="924" height="387" alt="image" src="https://github.com/user-attachments/assets/24938ff6-28b1-4720-9270-d4723e2f10e8" />


And those lines spelled something.

From left to right, across the graph, the path traced out letters:

```
KICTF{0FF_7h3_r4d4r_n0w}
```

That was the “big picture.”

The attacker literally drew the flag using GPS coordinates.

---

# Final Flag

```
KICTF{0FF_7h3_r4d4r_n0w}
```

---

## Why This Challenge Was Clever

This wasn’t about deep protocol exploitation or encryption.

It tested whether you would:

* Notice injected spoofed data
* Validate checksums
* Convert coordinate formats correctly
* And most importantly… visualize the data

The attacker told us exactly what to do:

> Reconstruct the stream.
> See the big picture.

Once you removed the noise and stepped back, the answer was right there — drawn across the map.

Simple idea. Clean execution. Very satisfying solve.
