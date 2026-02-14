# DTMF - Writeup

<img width="522" height="698" alt="image" src="https://github.com/user-attachments/assets/4d524f2c-8acd-4d08-ae3c-4242e9bcff20" />


## Step 1 — Extract DTMF Tones

DTMF tones correspond to digits pressed on a phone keypad.  
We can decode them using **multimon-ng**.

### Command

`multimon-ng -t wav -a DTMF message.wav`

### Output

This produced a sequence of digits representing keypad presses, which corresponded to binary values.

<img width="1880" height="454" alt="image" src="https://github.com/user-attachments/assets/ca0c0281-d7a1-4180-9d3d-c2c4b38f33e2" />

---

## Step 2 — Convert Binary → Text

The extracted digits formed a binary string.

Using **CyberChef**:

1. Input the binary string
    
2. Apply **From Binary**
    
3. Result → Base64 string
    

---

## Step 3 — Decode Base64

Applying **From Base64** in CyberChef revealed the ciphertext:

`0rmgj{Tu1m1_b4h_isc5_vntr}`

This clearly resembles a flag but does not match the expected format.


<img width="1497" height="595" alt="image" src="https://github.com/user-attachments/assets/868c329c-9349-49dc-945e-e95ee1fd8b26" />


Known flag format:

`0xfun{...}`

So the string is encrypted.

---

## Step 4 — Inspect File Metadata

Always check metadata in forensic challenges.

### Command

`exiftool message.wav`

### Important Output

`Comment : uhmwhatisthis`

This unusual string is likely a cryptographic key.

---

## Step 5 — Identify Cipher

Compare ciphertext prefix with expected flag prefix:

`Cipher: 0 r m g j { Target: 0 x f u n {`

Observations:

- Numbers and symbols unchanged
    
- Letters substituted
    
- Not a fixed shift → Not Caesar
    
- Likely polyalphabetic substitution → **Vigenère Cipher**
    

The metadata comment is a perfect candidate for the key:

`Key: uhmwhatisthis`

---

## Step 6 — Decrypt Using Vigenère

Non-alphabet characters are ignored during decryption.

### Python Solver

``` 
def solve_vigenere(ciphertext, key):
    key = key.lower()
    plaintext = []
    key_index = 0

    for char in ciphertext:
        if char.isalpha():
            # Determine shift based on key character
            shift = ord(key[key_index % len(key)]) - ord('a')
            
            # Handle uppercase
            if char.isupper():
                base = ord('A')
                decrypted_char = chr((ord(char) - base - shift) % 26 + base)
            # Handle lowercase
            else:
                base = ord('a')
                decrypted_char = chr((ord(char) - base - shift) % 26 + base)
            
            plaintext.append(decrypted_char)
            key_index += 1
        else:
            # Non-alpha characters (0, 1, 4, 5, {, }, _) are passed through
            plaintext.append(char)

    return "".join(plaintext)

cipher = "0rmgj{Tu1m1_b4h_isc5_vntr}"
key_phrase = "uhmwhatisthis"

print(f"Flag: {solve_vigenere(cipher, key_phrase)}")
```
---

## Step 7 — Final Result

Decrypted text:

`0xfun{Mu1t1_t4p_plu5_dtmf}`


---

# Final Flag

`0xfun{Mu1t1_t4p_plu5_dtmf}`

---
