# The Moving Statue

## Challenge identity

- **Points:** 250
- **Category:** Forensics primary, with Web and Cryptography; inferred
- **Status:** **PARTIALLY SOLVED / BROKEN DISTRIBUTION**
- **Confirmed flag:** None

## Original question

> **THE MOVING STATUE**  
> **250**  
> 0 0
>
> A paranoid curator is dead, and he took his secret to the grave --- almost. He shattered the vault key and hid the shards among 200 museum photographs, then salted the collection with fakes to bury the truth in noise. One clue survives: he only ever trusted what came from home.The Reading-Room Desk still stands. Answer to it, read the room, and open the vault.
>
> https://gallery-vault-desk.onrender.com/
>
> https://drive.google.com/file/d/1glF6F5Bog-UG2W1zNMzUmP2PUgQJ0FHR/view?usp=sharing

## High-level overview

Each of the 200 museum images contained an `SSSA1` Shamir share over GF(256). Most were decoys. The Reading-Room Desk first required three Point Blank trivia answers. Correctly answering them revealed the selection rule: trust photographs of Indian sculptures.

Eight images matched that provenance rule. Their shares were accepted by the live reconstruction endpoint, which returned an AES-256 key. The frontend then expected `vault.enc`, formatted as a 12-byte AES-GCM nonce followed by ciphertext and tag.

The supplied archive contained exactly 200 images plus `NOTE.txt`; `vault.enc` was missing. ZIP structure, comments, image metadata, trailers, public sources, and likely server paths were checked. The final flag could not be recovered from the distributed package.

## Artifact integrity

Archive SHA-256:

```text
FA25CEF690A7ABDDD320910C5845FB063E2328D6EF5F293E8356F9E95BA135E9
```

Archive contents:

```text
200 images
NOTE.txt
```

Missing required artifact:

```text
vault.enc
```

## Step-by-step procedure

### 1. Download and verify the archive

Google Drive ID:

```text
1glF6F5Bog-UG2W1zNMzUmP2PUgQJ0FHR
```

Windows verification:

```powershell
Get-FileHash .\gallery_download -Algorithm SHA256
```

Expected hash:

```text
FA25CEF690A7ABDDD320910C5845FB063E2328D6EF5F293E8356F9E95BA135E9
```

### 2. Solve the Reading-Room Desk quiz

The desk asked:

1. How many Point Blank members cracked GSoC in 2026?
2. Where did Shubhang Sinha attend his ACM Summer School 2026?
3. Where does Tanmay K currently work?

Answers:

```text
16
IISc Bangalore
Aspora
```

The frontend sent MCQ option indexes:

```json
{"answers":[16,1,2]}
```

Example:

```bash
curl -s https://gallery-vault-desk.onrender.com/api/submit \
  -H "Content-Type: application/json" \
  --data '{"answers":[16,1,2]}'
```

The desk then disclosed that the genuine photographs were Indian sculptures.

### 3. Extract every metadata share

The valid token pattern was:

```regex
SSSA1:[0-9a-fA-F]{2}:[0-9a-fA-F]{64}:[0-9a-fA-F]{4}
```

The extractor had to inspect:

- Ordinary Pillow image metadata
- Top-level EXIF
- GPS IFD `0x8825`
- EXIF sub-IFD `0x8769`
- UTF-8 and UTF-16LE values

Run:

```powershell
python "C:\Users\kingg\Documents\codex 2\moving-statue\extract_shares.py"
```

Control output:

```text
images=200 shares=200
indian_sculptures=8
```

### 4. Select the eight authentic shares

```text
IMG_2433.jpg
SSSA1:06:c918f4f9a95e307b762e1fa324926986fa8382d9dd8c43f93bb3c0c0052ac28e:7249

IMG_5598.jpg
SSSA1:01:7665d9b15fa7839c4fd20d1506d3f65eb5e3a49888315183632c2f85bad84975:111e

IMG_6734.jpg
SSSA1:02:7510a267046131376ce086ca8efbe394b1b32ee074018ddfb915ecadc3d9b5f9:5150

museum_2026_0548.jpg
SSSA1:08:266632402811a25ba8988b849abdc1ecc3d633c8a31f5183942e5df5dece211b:9892

museum_2026_0874.png
SSSA1:04:05e7b548f8b5489528175f51862e073d7d219a1db436f23b52e71bc58eb5d7db:0d4e

museum_2026_0884.jpg
SSSA1:07:0692ce9ea373fa3e0b25d48e0e0612f77971106548062e6788ded8edf7b42b57:7a03

scan82069.jpg
SSSA1:05:ca6d8f2ff29882d0551c947cacba7c4cfed308a121bc9fa5e18a03e87c2b3e02:5691

sculpture_250.jpg
SSSA1:03:ba9a98000e4cfb7211eb4de7a46f98e53241bc5ce18be0410a78f48031475c20:2483
```

### 5. Reconstruct the AES key

The frontend posts newline-delimited shares:

```json
{"shares":"SSSA1:...\nSSSA1:..."}
```

Endpoint:

```text
POST /api/reconstruct
```

Any three authentic fragments should meet the threshold. All eight were submitted as a stronger control. The server returned:

```text
b9efe3d6558a49d932d9c6382c478d2f361136241dbb3c1dd04137a84846a0ac
```

This is an AES-256 key, not a flag.

### 6. Attempt the intended final decryption

The frontend defines:

```text
vault.enc = 12-byte nonce || ciphertext || GCM tag
```

Expected command:

```powershell
python -c "from cryptography.hazmat.primitives.ciphers.aead import AESGCM;b=open('vault.enc','rb').read();print(AESGCM(bytes.fromhex('b9efe3d6558a49d932d9c6382c478d2f361136241dbb3c1dd04137a84846a0ac')).decrypt(b[:12],b[12:],None).decode())"
```

This step is blocked because `vault.enc` is absent.

## Skills and workflows used

- `solve-challenge`
- `ctf-forensics`
- `ctf-web`
- OSINT-style public-source research

## Tools, libraries, and services used

- Python 3
- Pillow/PIL
- Regular expressions
- EXIF/GPS sub-IFD parsing
- PowerShell ZIP and hash inspection
- Google Drive
- Render-hosted Reading-Room Desk
- HTTP/API inspection
- Web search
- SHA-256
- Shamir Secret Sharing over GF(256)
- `cryptography.hazmat.primitives.ciphers.aead.AESGCM`

## Evidence and controls

- Exactly 200 images were parsed.
- Exactly 200 unique valid-format shares were found.
- Exactly eight descriptions matched the Indian-sculpture provenance rule.
- The live desk accepted the quiz answers.
- The official reconstruction endpoint returned the AES key.
- The archive hash was recorded.
- No `vault.enc` appeared in the ZIP directory, comments, embedded structures, metadata, or trailers.
- No public challenge source supplied the missing ciphertext.

## Surviving files

```text
C:\Users\kingg\Documents\codex 2\moving-statue\extract_shares.py
C:\Users\kingg\Documents\codex 2\moving-statue\extracted\NOTE.txt
C:\Users\kingg\Documents\codex 2\moving-statue\questions.json
C:\Users\kingg\Documents\codex 2\moving-statue\app.js
C:\Users\kingg\Documents\codex 2\moving-statue\gallery_download
C:\Users\kingg\Documents\codex 2\moving-statue\extracted\images
```

## Warning: deliberate decoy flags

These 13 strings were planted in photographs and must not be submitted:

```text
pbctf{31gh7fold_vault_r3f0rged}
pbctf{54cred_4rchive_un5eal3d}
pbctf{anc13n7_g4llery_r3f0rged}
pbctf{br0k3n_fragm3nts_re57ored}
pbctf{br0ken_1d0ls_un5e4led}
pbctf{e1ghtfold_mo5a1c_reuni7ed}
pbctf{f0rg0773n_curator_r3jo1ned}
pbctf{frac7ured_v4ult_r3join3d}
pbctf{g1ld3d_n4tar4j4_4553mbl3d}
pbctf{h1dden_bronz3_rev34l3d}
pbctf{m0lt3n_cur4t0r_r3join3d}
pbctf{mol7en_1d0l5_r3s7ored}
pbctf{whi5p3r3d_1d0ls_rec0v3r3d}
```

## Diagram

A separate Excalidraw diagram is unnecessary; the missing-artifact boundary is compact:

```text
200 photos → 200 shares → quiz/provenance filter → 8 real shares
     → /api/reconstruct → AES-256 key → vault.enc required
                                              └── missing
```
