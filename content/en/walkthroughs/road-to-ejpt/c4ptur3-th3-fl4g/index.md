---
title: "c4ptur3-th3-fl4g"
date: 2026-01-01
draft: false
description: "Walkthrough of c4ptur3-th3-fl4g from TryHackMe. Encoding translations, spectrograms and steganography — eJPT preparation."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "ctf",
    "encoding",
    "steganography",
    "cryptography",
    "base64",
    "binary",
    "morse",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 7
---

## Summary

**c4ptur3-th3-fl4g** is the seventh machine of the _Road to eJPTv2_ series and the most different one so far. No service exploitation, no reverse shells, no privesc. It's a pure **encoding, cryptography and steganography** challenge — designed to get you comfortable with data representation systems that appear constantly in CTFs and forensic analysis.

This room covers: leetspeak, binary, Base32, Base64, hexadecimal, ROT13, ROT47, Morse code, BCD, Brainfuck/Malbolge, audio spectrograms and image steganography.

| Attribute      | Value                                                         |
| -------------- | ------------------------------------------------------------- |
| **Platform**   | TryHackMe                                                     |
| **Difficulty** | Easy                                                          |
| **OS**         | N/A (encoding challenge)                                      |
| **Room**       | [c4ptur3-th3-fl4g](https://tryhackme.com/room/c4ptur3th3fl4g) |
| **Skills**     | Encoding/Decoding, Steganography, Spectrograms, Basic OSINT   |

### 🎥 Video Walkthrough

{{< youtube yvhr_lWnCnQ >}}

> If you prefer to follow the walkthrough step by step, keep reading. The video covers the same process in visual format.

### Tools Used

- [CyberChef](https://gchq.github.io/CyberChef/) — swiss army knife for encoding/decoding
- [Audacity](https://www.audacityteam.org/) — spectrogram visualization
- `steghide` — extracting hidden data from images
- `binwalk` — analysis of composite files

### Solution Overview

This room is divided into four sections:

1. **Translations & Shifting** — 10 encoding/decoding challenges
2. **Spectrograms** — hidden message in audio frequencies
3. **Steganography** — data hidden inside an image
4. **Security through obscurity** — file inside another file

---

## Section 1: Translations & Shifting

This section presents 10 encoded strings that need to be decoded. The main tool is **CyberChef** (https://gchq.github.io/CyberChef/), which allows applying multiple transformations in a chain.

### Challenge 1: Leetspeak

`c4n y0u c4p7u23 7h3 f149?`

**Leetspeak** (l33tspeak) is a substitution of letters with visually similar numbers or symbols. Common in hacker culture since the 80s.

| Leetspeak | Letter |
| --------- | ------ |
| 4         | a      |
| 0         | o      |
| 7         | t      |
| 2         | r      |
| 3         | e      |
| 1         | i      |
| 9         | g      |

> **Answer:** `can you capture the flag?`

### Challenge 2: Binary

```
01101100 01100101 01110100 01110011 00100000 01110100 01110010 01111001 00100000 01110011 01101111 01101101 01100101 00100000 01100010 01101001 01101110 01100001 01110010 01111001 00100000 01101111 01110101 01110100 00100001
```

Each group of 8 bits represents an ASCII character. In CyberChef: `From Binary`.

> **Answer:** `lets try some binary out!`

### Challenge 3: Base32

```
MJQXGZJTGIQGS4ZAON2XAZLSEBRW63LNN5XCA2LOEBBVIRRHOM======
```

Base32 uses the A-Z and 2-7 alphabet, with `=` as padding. In CyberChef: `From Base32`.

> **Answer:** `base32 is super common in CTF's`

### Challenge 4: Base64

```
RWFjaCBCYXNlNjQgZGlnaXQgcmVwcmVzZW50cyBleGFjdGx5IDYgYml0cyBvZiBkYXRhLg==
```

Base64 uses A-Z, a-z, 0-9, +, / and `=` as padding. Recognizable by the `==` at the end. In CyberChef: `From Base64`.

> **Answer:** `Each Base64 digit represents exactly 6 bits of data.`

### Challenge 5: Hexadecimal

```
68 65 78 61 64 65 63 69 6d 61 6c 20 6f 72 20 62 61 73 65 31 36 3f
```

Each hex pair represents one byte. `20` is the space character in ASCII. In CyberChef: `From Hex`.

> **Answer:** `hexadecimal or base16?`

### Challenge 6: ROT13

```
Ebgngr zr 13 cynprf!
```

ROT13 shifts each letter 13 positions in the alphabet. Applied twice it returns to the original (it is its own inverse). In CyberChef: `ROT13`.

> **Answer:** `Rotate me 13 places!`

### Challenge 7: ROT47

```
*@F DA:? >6 C:89E C@F?5 323J C:89E C@F?5 Wcf E:>6DX
```

ROT47 is like ROT13 but applies to 94 printable ASCII characters (from 33 to 126), not just letters. That's why it can encrypt symbols, numbers and letters. In CyberChef: `ROT47`.

> **Answer:** `You spin me right round baby right round (47 times)`

### Challenge 8: Morse Code

```
. .-.. . -.-. --- -- -- ..- -. .. -.-. .- - .. --- -.
. -. -.-. --- -.. .. -. --.
```

Morse code uses dots and dashes to represent letters. In CyberChef: `From Morse Code`.

> **Answer:** `TELECOMMUNICATION ENCODING`

### Challenge 9: BCD (Binary Coded Decimal)

```
85 110 112 97 99 107 32 116 104 105 115 32 66 67 68
```

BCD represents each decimal digit with its bit equivalent. These values are directly decimal ASCII codes. In CyberChef: `From Charcode` (decimal base).

> **Answer:** `Unpack this BCD`

### Challenge 10: Multi-layer string

```
LS0tLS0gLi0tLS0g...
```

This string requires decoding in multiple layers:

1. Base64 → produces Morse code
2. Morse → produces plain text

In CyberChef: `From Base64` → `From Morse Code`.

> **Answer:** `Let's make this a bit trickier...`

---

## Section 2: Spectrograms

A **spectrogram** is a visual representation of the frequencies of an audio signal over time. Messages can be hidden in audio by setting the volume of specific frequencies to form letters or images when viewed in a spectrogram.

### Process

1. Download the audio file from the room
2. Open it in **Audacity**
3. In the audio track, click on the track name → select **"Spectrogram"**
4. The spectrogram will reveal text visible in the frequencies

> **Answer:** `Super Secret Message`

---

## Section 3: Steganography

**Steganography** hides information inside other files — images, audio, video — in a way that is not visible to the naked eye. Unlike cryptography (which encrypts the message), steganography hides the existence of the message.

### Process

1. Download the image from the room
2. Use `steghide` to extract hidden data:

```bash
steghide extract -sf image.jpg
```

3. If it has a passphrase, try with an empty string (press Enter without typing anything)
4. The extracted file contains the answer

> **Answer:** `SpaghettiSteg`

---

## Section 4: Security through obscurity

This section shows how attackers (and defenders) can hide files inside other files — a technique used in both malware and CTFs.

### Challenge 1: File inside a file

1. Download the file from the room
2. Use `binwalk` to see what's inside:

```bash
binwalk file
```

3. Extract the content:

```bash
binwalk -e file
```

4. Inside you will find the hidden file

> **First hidden file:** `hackerchat.png`

### Challenge 2: Hidden text inside the file

1. With the extracted file, inspect it:

```bash
strings hackerchat.png
```

Or open it with a hex editor and look for plain text at the end of the file.

> **Hidden text:** `AHH_YOU_FOUND_ME!`

---

## Quick Encoding Reference

This table summarizes the most common encodings in CTFs:

| Encoding    | Visual characteristic           | Tool                   |
| ----------- | ------------------------------- | ---------------------- |
| Binary      | Only 0s and 1s in groups of 8   | CyberChef: From Binary |
| Hexadecimal | Characters 0-9 and a-f in pairs | CyberChef: From Hex    |
| Base32      | Uppercase + 2-7, padding with = | CyberChef: From Base32 |
| Base64      | A-Z, a-z, 0-9, +/, padding ==   | CyberChef: From Base64 |
| ROT13       | Readable text but shifted       | CyberChef: ROT13       |
| ROT47       | Symbols mixed with text         | CyberChef: ROT47       |
| Morse       | Dots, dashes and spaces         | CyberChef: From Morse  |
| Leetspeak   | Numbers mixed with letters      | Manual inspection      |

---

## Lessons Learned

- **CyberChef is indispensable for CTFs** — Most of the encodings in this room are solved in seconds with CyberChef. Learning to chain operations in CyberChef is a fundamental skill for any CTF player.
- **Recognizing encodings at a glance saves time** — `==` at the end → Base64. Uppercase with 2-7 → Base32. Dots and dashes → Morse. 0s and 1s in groups of 8 → Binary. Building this visual recognition is essential.
- **Steganography isn't just images** — Information can be hidden in audio (spectrograms), video, PDF documents, ZIP files, and more. Whenever you have a file in a CTF, ask yourself: is there something hidden here?
- **`binwalk` and `strings` are your first commands with unknown files** — `strings` shows readable text inside any binary file. `binwalk` detects embedded files. Use them before opening any suspicious file.
- **Security through obscurity is not real security** — Hiding files inside others or changing extensions does not protect data. An attacker with `binwalk` or `strings` finds it in seconds. Real security requires encryption, not concealment.

### For the eJPT

Although the eJPT focuses more on network and system exploitation, the concepts from this room are relevant for:

- Recognizing encoded data in web application responses (Base64 in cookies, JWT tokens, etc.)
- Basic forensic analysis in post-exploitation
- Understanding how information is hidden in files

**Approximate completion time:** 30-45 minutes with CyberChef open and the encoding reference at hand.

---

## References

- [c4ptur3-th3-fl4g — TryHackMe](https://tryhackme.com/room/c4ptur3th3fl4g)
- [CyberChef — GCHQ](https://gchq.github.io/CyberChef/)
- [Audacity](https://www.audacityteam.org/)
- [steghide documentation](http://steghide.sourceforge.net/)
- [binwalk documentation](https://github.com/ReFirmLabs/binwalk)
- [GTFOBins](https://gtfobins.github.io/)
