# Dead Frequency

**Category:** Forensics
**Flag:** `KICTF{t1m1ng_is_3v3ryth1ng_and_dns_n3v3r_li3s}`

> *Our IDS flagged unusual ping activity from an internal workstation to an external IP. We captured the traffic but nothing looks obviously malicious. The analyst swears he sees the machine "talking back" through DNS. Figure out what was stolen.*

---

When I first read the challenge title — **Dead Frequency** — I knew it wasn’t random. In CTFs, titles are rarely decorative. They’re hints.

“Dead” and “Frequency” turned out to be literal clues hiding in plain sight.

---

## Opening the Capture

We’re given a `capture.pcapng` file. I loaded it into Wireshark to get a high-level overview.

![Packet Overview](https://github.com/user-attachments/assets/378bf94d-00e5-4302-ab0f-01442ef2f570)

There are **3,374 packets** total:

* **ICMP** — 1,384 packets (a *lot* of pings)
* **DNS** — 72 packets
* **TCP/443** — ~1,900 packets (normal HTTPS traffic)

At first glance, it looks like a normal workstation (`192.168.1.105`) browsing Google, Reddit, Discord, YouTube. Nothing obviously malicious.

But the ping volume stood out.

---

## The Strange Pings

Filtering for ICMP traffic showed something interesting.

The first few pings are completely normal — the machine pinging its gateway (`192.168.1.1`) with standard payloads like:

```
ABCDEFGHIJKLMNOP
```

![Normal ICMP](https://github.com/user-attachments/assets/cfda470b-d901-4ec6-935a-85f01c20f2a9)

Then something changes.

An external IP — `203.0.113.10` (TEST-NET-3 range) — starts sending ICMP packets *into* the internal machine. That’s already unusual.

And the payloads?

They all start with:

```
DE AD
```

![Suspicious ICMP](https://github.com/user-attachments/assets/eba0e56d-d899-4f93-984a-ace029ac7e02)

The challenge is called **Dead** Frequency. The packets literally start with `DE AD`. That’s not coincidence.

Looking closer, the structure is:

```
DE AD [2-byte index] [32 bytes of data]
```

Most packets contain repeating ASCII patterns like:

```
D34DC2C2D34DC2C2D34DC2C2D34DC2C2
```

That repetition looked intentional — like a key.

---

## XOR — The Hidden Layer

If most packets contain the same repeating data and a few differ, that screams **XOR masking**.

It looked like:

* The 32-byte payload was XOR’d with `D34DC2C2` (repeating)
* Packets that decode to all zeros are noise
* Packets that decode to real bytes contain hidden data

So I wrote a quick Python script to:

* Extract ICMP payloads from `203.0.113.10`
* Remove the `DEAD` prefix
* Use the next two bytes as a chunk index
* XOR the remaining 32 bytes with `D34DC2C2`

The very first meaningful decode gave:

```
7f 45 4c 46 02 01 01 00 ...
```

That’s an **ELF header**.

They were exfiltrating a Linux binary over ICMP ping payloads.

---

## Reassembling the Binary

The two bytes after `DEAD` form a **16-bit big-endian chunk index**. That tells us the correct order of chunks.

There were 450 chunks in total.

After:

* Sorting by index
* XOR decoding each chunk
* Concatenating them

I reconstructed a clean **14,400-byte x86-64 ELF binary**.

Inside the binary, running `strings` revealed critical clues:

```
B32: %s
attacker-ns.io
YBNDRFG8EJKMCPQXOT1UWISZA345H769
```

Three important discoveries:

| String                             | Meaning                    |
| ---------------------------------- | -------------------------- |
| `B32: %s`                          | Data is encoded in Base32  |
| `attacker-ns.io`                   | Command-and-control domain |
| `YBNDRFG8EJKMCPQXOT1UWISZA345H769` | Custom z-base-32 alphabet  |

The binary itself was the exfiltration tool — delivered through ICMP, then used to steal data via DNS.

The analyst was right.

---

## Following the DNS Trail

Back to the PCAP. Filtering for DNS queries revealed:

![DNS Queries](https://github.com/user-attachments/assets/09c8ee72-6817-49a4-a03c-498a608d72bd)

```
00-epkrc65wgfsun5u8m7wz.attacker-ns.io
01-gz3uqa3zr6mwpyash349.attacker-ns.io
02-cfzgez5rp33i65tuqa3z.attacker-ns.io
03-rz5cpr3zg9e.attacker-ns.io
data-q1rge2xvbf9uawnlx3ryev90aglzx2lzx2zha2v9.attacker-ns.io
```

The numbering (`00-` to `03-`) tells us the data is chunked.

The encoding matches the custom alphabet found in the binary.

This is classic **DNS exfiltration** — encoding stolen data inside subdomain labels and querying an attacker-controlled nameserver.

Since DNS is rarely blocked, it’s the perfect covert channel.

---

## Decoding the Data

The binary uses **z-base-32**, which has a custom alphabet:

```
Standard: ABCDEFGHIJKLMNOPQRSTUVWXYZ234567
Custom:   YBNDRFG8EJKMCPQXOT1UWISZA345H769
```

To decode:

* Translate custom alphabet → standard Base32
* Decode normally

After reconstructing and decoding the subdomains, the output was:

```
CTF{t1m1ng_is_3v3ryth1ng_and_dns_n3v3r_li3s}
```

With the event prefix:

# Flag

```
KICTF{t1m1ng_is_3v3ryth1ng_and_dns_n3v3r_li3s}
```

![Flag Output](https://github.com/user-attachments/assets/cebeda6f-c82d-4f8c-bedc-c16b1c72731d)

---

## What This Challenge Really Showed

This wasn’t just about packet inspection.

It demonstrated:

* ICMP as a covert file transfer channel
* XOR-based obfuscation
* DNS-based data exfiltration
* Custom Base32 encoding
* Multi-stage malware delivery

The title said it all.

**Dead** → `DE AD`
**Frequency** → ICMP pings

And in the end, timing and DNS never lie.
