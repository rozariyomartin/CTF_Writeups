# Dead Frequency

**Category:** Forensics
**Flag:** `KICTF{t1m1ng_is_3v3ryth1ng_and_dns_n3v3r_li3s}`

*Our IDS flagged unusual ping activity from an internal workstation to an external IP. We captured the traffic but nothing looks obviously malicious. The analyst swears he sees the machine 'talking back' through DNS. Figure out what was stolen.*

Alright, so we've got a PCAP with "unusual pings" and some analyst who thinks DNS is involved. Two words in the challenge title stood out to me right away — **"Dead"** and **"Frequency"** — and as we'll see, both turned out to be literal clues hiding in plain sight.

---

## First Look — Opening the Capture

We're handed `capture.pcapng`. I loaded it up in Wireshark and the first thing I did was get a high-level overview of what's going on.

<img width="951" height="390" alt="image" src="https://github.com/user-attachments/assets/378bf94d-00e5-4302-ab0f-01442ef2f570" />

The capture has **3,374 packets** — a mix of:

- **ICMP** (1,384 packets) — that's a LOT of pings
- **DNS** (72 packets) — mostly normal-looking queries
- **TCP/443** (~1,900 packets) — regular HTTPS noise

At a glance, it looks like someone's workstation (`192.168.1.105`) doing normal stuff — browsing Google, Reddit, Discord, YouTube. Nothing screams "malicious"... at first.

---

## Something's Off With the Pings

I filtered for ICMP and noticed something immediately. The first few packets are perfectly normal — the workstation pings its gateway `192.168.1.1` with standard 16-byte payloads containing `ABCDEFGHIJKLMNOP`:

<img width="1903" height="1018" alt="image" src="https://github.com/user-attachments/assets/cfda470b-d901-4ec6-935a-85f01c20f2a9" />

Normal ping. Nothing to see here. But then comes a **different** source IP — `203.0.113.10` — an external address from the TEST-NET-3 range, pinging *into* the internal network. And the payloads are... weird.

<img width="1891" height="1012" alt="image" src="https://github.com/user-attachments/assets/eba0e56d-d899-4f93-984a-ace029ac7e02" />

Every single one of these packets starts with the bytes **`DE AD`**. The challenge is literally called "**Dead** Frequency." That's our first breadcrumb.

Looking closer at the payload structure:

```
DE AD [byte3] [byte4] [32 bytes of data]
```

The 32-byte data portion looks semi-random, but I noticed a repeating pattern in most packets: `D34DC2C2D34DC2C2D34DC2C2D34DC2C2` — the ASCII string **"D34DC2C2"** repeated four times. A few packets deviate from this pattern though, and deviations scream "hidden data."

---

## XOR'ing the ICMP Payloads

If most packets contain `D34DC2C2` repeated, and some don't... it smells like the data has been **XOR'd with `D34DC2C2` as the key**. Packets that XOR to all-zeros are just cover traffic. Packets that produce non-zero results carry actual hidden data.

So I wrote a quick Python script to:

1. Extract all ICMP payloads from `203.0.113.10`
2. Strip the `DEAD` prefix and the two index bytes
3. XOR the remaining 32 bytes with the repeating key `D34DC2C2`

And the very first non-zero result made my jaw drop:

```
7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
```

**That's an ELF header.** They're exfiltrating a full Linux binary over ICMP ping payloads! 

## Reassembling the Binary

The two bytes after `DEAD` (byte3 and byte4) form a **16-bit chunk index** — telling us the order to reassemble the pieces. I extracted all 450 chunks, sorted them by index (0 through 449), XOR'd each with the key, and concatenated the results.

Out popped a clean **14,400-byte x86-64 ELF binary**. 

```python
# Key insight: bytes 2-3 form a 16-bit big-endian chunk index
chunk_idx = (payload[2] << 8) | payload[3]
key = b"D34DC2C2"
decoded = bytes(b ^ key[i % 8] for i, b in enumerate(payload[4:]))
```

I dumped the strings from the reconstructed binary and found gold:

```
B32: %s
attacker-ns.io
kworker/u:2
Mozilla/5.0 (X11; Linux x86_64) systemd/249
YBNDRFG8EJKMCPQXOT1UWISZA345H769
GCC: (GNU) 15.2.1 20260209
```

Three things matter here:

| String                             | What it tells us                     |
| ---------------------------------- | ------------------------------------ |
| `B32: %s`                          | The binary encodes data using Base32 |
| `attacker-ns.io`                   | The C2 domain for DNS exfiltration   |
| `YBNDRFG8EJKMCPQXOT1UWISZA345H769` | A **custom z-base-32 alphabet**      |

So the binary is the **exfiltration tool itself** — it was delivered to the workstation via the ICMP covert channel and then used DNS to steal data. The analyst was right all along.

---

## The DNS Trail

Going back to the PCAP, I filtered for DNS queries and found the smoking gun — 5 queries to `attacker-ns.io`, buried between normal browsing traffic:

<img width="1886" height="1012" alt="image" src="https://github.com/user-attachments/assets/09c8ee72-6817-49a4-a03c-498a608d72bd" />

```
00-epkrc65wgfsun5u8m7wz.attacker-ns.io
01-gz3uqa3zr6mwpyash349.attacker-ns.io
02-cfzgez5rp33i65tuqa3z.attacker-ns.io
03-rz5cpr3zg9e.attacker-ns.io
data-q1rge2xvbf9uawnlx3ryev90aglzx2lzx2zha2v9.attacker-ns.io
```

The subdomain labels are **numbered** (`00-` through `03-`) — that's the exfiltrated data, chunked and ordered. The `data-` label is probably metadata. And the encoding? It matches that custom alphabet from the binary.

Classic DNS exfiltration — data is encoded into subdomain labels and "queried" to an attacker-controlled nameserver. Since DNS is almost never blocked, it's a perfect covert channel.

---

## Cracking the Encoding

The binary uses **z-base-32** — a human-oriented variant of Base32 with a custom alphabet. The standard Base32 alphabet is `ABCDEFGHIJKLMNOPQRSTUVWXYZ234567`, but this binary uses:

```
Standard:  ABCDEFGHIJKLMNOPQRSTUVWXYZ234567
Custom:    YBNDRFG8EJKMCPQXOT1UWISZA345H769
```

To decode, I built a translation table from the custom alphabet back to standard Base32, then decoded normally:

```python
import base64

custom = "ybndrfg8ejkmcpqxot1uwisza345h769"
std    = "abcdefghijklmnopqrstuvwxyz234567"
table  = str.maketrans(custom, std)

labels = [
    "epkrc65wgfsun5u8m7wz",
    "gz3uqa3zr6mwpyash349",
    "cfzgez5rp33i65tuqa3z",
    "rz5cpr3zg9e"
]

combined = ''.join(labels).translate(table).upper()
combined += '=' * ((8 - len(combined) % 8) % 8)

flag = base64.b32decode(combined).decode()
print(flag)
```

**Output:**

```
CTF{t1m1ng_is_3v3ryth1ng_and_dns_n3v3r_li3s}
```

Adding the event prefix → **`KICTF{t1m1ng_is_3v3ryth1ng_and_dns_n3v3r_li3s}`** 🎉

<img width="500" height="105" alt="image" src="https://github.com/user-attachments/assets/cebeda6f-c82d-4f8c-bedc-c16b1c72731d" />

