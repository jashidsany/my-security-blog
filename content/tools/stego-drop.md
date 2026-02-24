---
title: "Building stego-drop: Hiding Shellcode in PNG Images with LSB Steganography"
date: 2026-02-24
draft: false
tags: ["steganography", "red-team", "python", "offensive-security", "malware-dev"]
categories: ["Tools"]
description: "A walkthrough of building stego-drop, a Python LSB steganography tool for embedding shellcode and binary payloads into PNG images."
---

Steganography is the practice of hiding data inside something that looks completely normal. Unlike encryption, which makes data unreadable, steganography makes data invisible. On a red team engagement, that distinction matters. Encrypted traffic gets flagged. A PNG image of a cat sitting on a keyboard? Nobody looks twice.

I built **stego-drop** to explore this concept hands-on: a Python tool that embeds binary payloads (shellcode, scripts, whatever you want) into PNG images using Least Significant Bit encoding. In this post I'll walk through how LSB steganography works, how I built the tool, and how to use it.

The full source is on my GitHub: [github.com/jashidsany/stego-drop](https://github.com/jashidsany/stego-drop)

---

## What is LSB Steganography?

Every pixel in a PNG image stores color as three channels: Red, Green, and Blue. Each channel is an 8-bit value ranging from 0 to 255. That means each channel looks something like this in binary:

```bash
Red:   10000110  (134)
Green: 11001000  (200)
Blue:  01001110  (78)
```

The **least significant bit** is the rightmost bit. Flipping it changes the value by 1 at most. A pixel with Red = 134 becomes Red = 135. That's a change of 0.4%. Your eyes cannot see it. A monitor cannot display it. But that one bit can store data.

If we flip the LSB of all three channels in a single pixel, we can store 3 bits per pixel. A 1920x1080 image has 2,073,600 pixels, giving us roughly **759 KB** of hidden storage. That's enough to hide hundreds of shellcode payloads.

---

## The Idea Behind stego-drop

The concept is simple:

1. Take a clean PNG image (the "cover" image)
2. Take a payload (shellcode, a script, a binary, any file)
3. Convert the payload to bits
4. Replace the LSB of each pixel channel value with a payload bit
5. Save the result as a new PNG

The output is a valid PNG image that looks identical to the original. But hidden in the noise floor of its pixel data is your payload, waiting to be extracted.

---

## How the Encoding Works

Here's the core logic. Take a payload byte like `0xEB` (the first byte of a typical x86 JMP instruction):

```bash
0xEB in binary: 11101011
```

We need 8 pixel channel values to store this one byte (1 bit per channel). Say the first few pixel values are:

```bash
Before:  134  200  78  45  190  222  100  88
LSBs:      0    0   0   1    0    0    0   0
```

We overwrite each LSB with our payload bits:

```bash
Payload: 1    1   1   0    1    0    1   1
After:  135  201  79  44  191  222  101  89
```

The maximum any single value changed is 1. Across a 1920x1080 image, a 71-byte shellcode payload modifies only 568 out of 6,220,800 channel values. That's 0.009%. The PSNR (Peak Signal-to-Noise Ratio) typically comes in above 80 dB, which means the images are statistically identical.

---

## Building stego-drop

I wanted the tool to be:

- **OS agnostic** - Python with Pillow and NumPy, runs everywhere
- **Flexible** - support different channel modes and bit depths
- **Cross-compatible** - optional stegano-compatible format for interop with other tools
- **Useful for analysis** - built-in steganalysis detection

### The Embedding Flow

The embed function flattens the image into a 1D array of channel values, then walks through it setting LSBs:

```python
img_array = np.array(cover_image)
flat = img_array.flatten()

for i, bit in enumerate(data_bits):
    flat[i] = (flat[i] & 0xFE) | bit  # Clear LSB, set to payload bit

stego_array = flat.reshape(img_array.shape)
```

`0xFE` is `11111110` in binary. ANDing with it zeros out the LSB. Then we OR in our payload bit. Clean and fast thanks to NumPy.

### Channel Modes

By default, bits are spread across all RGB channels sequentially. But you can target a single channel:

```bash
# Hide data only in the blue channel
python3 stego_drop.py embed -i cover.png -p payload.bin -o stego.png --mode b
```

Why would you do this? Some basic steganalysis tools only check the red or green channels. Blue channel modifications are also the least perceptible to human vision. It's a small OPSEC advantage.

### Multi-Bit Depth

For larger payloads, you can use more than 1 LSB per channel:

```bash
# Use 2 LSBs per channel (double capacity, slightly more detectable)
python3 stego_drop.py embed -i cover.png -p payload.bin -o stego.png --bits 2
```

With 2-bit encoding, each channel value can change by up to 3 instead of 1. With 4-bit, up to 15. The capacity scales linearly but detectability increases. For most red team use cases, stick with 1-bit.

### Stegano Cross-Compatibility

I added a `--stegano` flag that encodes text using the same format as the popular `stegano` Python library. This means you can embed with stego-drop and extract with stegano, or vice versa:

```bash
# Embed with stego-drop
python3 stego_drop.py embed -i cover.png -p message.txt -o stego.png --stegano

# Extract with stegano (third-party tool)
stegano-lsb reveal -i stego.png
```

The stegano format prepends a `length:message` header, so extraction auto-detects the message size. This is text-only though. Binary payloads like shellcode use the raw mode, which requires knowing the byte count on extraction.

---

## Testing It: Shellcode in a Cat Photo

I created a safe test shellcode (71 bytes, x86_64 Linux) that prints "STEGO-DROP PAYLOAD EXECUTED" and exits. Here's the full workflow:

```bash
# Check image capacity
$ python3 stego_drop.py capacity -i cat.png

  [*] Dimensions: 800x800 (640,000 pixels)
  [+] Max payload capacity: 234,692 bytes (229.2 KB)

# Embed the shellcode
$ python3 stego_drop.py embed -i cat.png -p shellcode.bin -o cat_stego.png

  [*] Payload: shellcode.bin (71 bytes)
  [*] Capacity: 234,692 bytes (0.0% utilized)
  [+] PSNR: 91.46 dB
  [+] PSNR > 50 dB - Visually identical.

# Extract on the other end
$ python3 stego_drop.py extract -i cat_stego.png --bytes 71 -o sc.bin

  [+] Extracted 71 bytes
  [+] Preview (hex):
      eb1e5e48c7c00100000048c7c70100000048c7c21c000000...
```

The stego image is a perfectly valid PNG. It opens in any image viewer. It looks identical to the original. But it's carrying shellcode.

---

## Built-in Steganalysis

stego-drop includes a `detect` command that runs basic steganalysis against suspect images. It checks four indicators per color channel:

- **LSB ratio** - balance of 0s and 1s in the LSB plane (embedded data tends toward 50/50)
- **Chi-square test** - measures histogram pair uniformity (embedding equalizes adjacent value pairs)
- **LSB entropy** - randomness of the LSB plane (embedded data has near-maximum entropy)
- **Transition rate** - sequential correlation of LSBs (natural images have correlated LSBs, embedded data doesn't)

```bash
$ python3 stego_drop.py detect -i suspect.png

  -- Red Channel --
    LSB 0/1 ratio:       0.4995 / 0.5005  (very balanced)
    Chi-square:          1470.52           (normal)
    LSB entropy:         0.999999          (near-maximum)
    LSB transition rate: 0.5000            (random-like)

  -- Verdict --
    HIGH probability of LSB steganography
    4/6 indicators triggered
```

This is basic analysis. Tools like stegdetect and aletheia are more sophisticated. But having detection built into the same tool is useful for understanding what your embedding looks like from the defender's perspective.

---

## Detection and OPSEC Considerations

LSB steganography is not bulletproof. Here's what to keep in mind:

**What works in your favor:**
- Visual inspection is useless. The pixel changes are below human perception.
- Small payloads in large images are statistically hard to detect. 71 bytes in a 1080p image modifies 0.009% of values.
- File size doesn't change anomalously. PNG compression varies naturally.

**What works against you:**
- If a defender has the original image, comparing it to the stego version reveals everything instantly.
- Heavy embedding (large payload relative to image size) is detectable through statistical analysis.
- JPEG conversion destroys the payload. Lossy compression overwrites the LSB data.
- Don't upload to platforms that re-encode images (Twitter, Instagram). Use platforms that preserve PNGs: Discord file attachments, email, direct transfer, or your own infrastructure.

---

## Try It

The tool is on GitHub with a full README, usage examples, and a safe test shellcode payload:

[github.com/jashidsany/stego-drop](https://github.com/jashidsany/stego-drop)

```bash
git clone https://github.com/jashidsany/stego-drop.git
cd stego-drop
pip install -r requirements.txt
python3 stego_drop.py capacity -i your_image.png
```

If you have questions or ideas for features, open an issue on the repo.

---

*Jashid Sany - [github.com/jashidsany](https://github.com/jashidsany)*
