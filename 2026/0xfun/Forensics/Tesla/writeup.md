# Tesla CTF - Writeup

## Challenge Overview
The challenge provided a file named [Tesla.sub](file:///home/martin/Downloads/ctf/2026/oxFun/Forensics/Tesla/Tesla.sub) (originally thought to be `data.bin` by the user, but identified as [Tesla.sub](file:///home/martin/Downloads/ctf/2026/oxFun/Forensics/Tesla/Tesla.sub)). This file mimics a Flipper Zero Sub-GHz capture file.

## Analysis
Upon inspecting [Tesla.sub](file:///home/martin/Downloads/ctf/2026/oxFun/Forensics/Tesla/Tesla.sub), we noticed the `RAW_Data` field did not contain standard timing data but rather space-separated 8-bit binary strings.

```
Filetype: Bad Usb 0xfun
Version: 1
Frequency: 433920000
Preset: FuriHalSubGhz
Protocol: RAW
RAW_Data: 11111111 11111110 00100110 ...
```

### Decoding Binary Data
Converting the binary strings to ASCII characters revealed an obfuscated Windows Batch script. The script used variable substring expansion (e.g., `%IlÃc:~42,1%`) to construct commands.

The variable `IlÃc` was defined as:
`set "IlÃc=pesbMUQl73oWnqD9rAvFRKZaf0hO5@dBN4uSzCtGjE YxITwXiVm1Jcgy26LkH8P"`

### Deobfuscation
Deobfuscating the script revealed the following key components:
1.  A PowerShell command that decodes the string: `'i could be something to this'`.
2.  A commented-out area containing a long hex string: `5958051a1b170013520746265a0e51435b36165752470b7f03591d1b364b501608616e`.
3.  A hint: `:: ive been encrypted many in ways::`.

### Decryption
We hypothesized that the hex string was XOR-encrypted using the phrase "i could be something to this" as the key.

Using a Python script to XOR the hex bytes with the key bytes revealed the flag.

## Solution Script
```python
import binascii

hex_string = "5958051a1b170013520746265a0e51435b36165752470b7f03591d1b364b501608616e"
key = "i could be something to this"

decrypted = bytearray()
key_bytes = key.encode('utf-8')
ciphertext = binascii.unhexlify(hex_string)

for i in range(len(ciphertext)):
    decrypted.append(ciphertext[i] ^ key_bytes[i % len(key_bytes)])

print(decrypted.decode('utf-8'))
```

## Flag
`0xfun{d30bfU5c473_x0r3d_w1th_k3y}`
