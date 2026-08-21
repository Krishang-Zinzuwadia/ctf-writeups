# Sh*t Secrets Got Exposed

## Challenge identity

- **Points:** 200
- **Category:** Forensics
- **Techniques:** QR reconstruction, steganography, OSINT, Git forensics
- **Status:** **CONFIRMED SOLVED**
- **Confirmed flag:** `pbctf{3f9d2c7a1e6b48d05c93a7f21e4d8b60}`
- **Diagram:** [`diagrams/shit-secrets-trail.excalidraw`](diagrams/shit-secrets-trail.excalidraw)

The supplied scoreboard independently lists this challenge as solved, category `Forensics`, value `200`.

## Original question

> **Sh\*t Secrets Got Exposed**  
> **200**  
> 0 0
>
> One of our engineers got careless. Our threat-intel team pulled four torn fragments of a QR code out of a shredder bin. Whatever it points to, they're convinced there's an exposed credential sitting at the end of the trail.
>
> https://drive.google.com/file/d/1JQl11pSQHCh7NA1IcJB7H043g489pyQU/view?usp=sharing

## Confirmed flag

```text
pbctf{3f9d2c7a1e6b48d05c93a7f21e4d8b60}
```

Underlying exposed credential:

```text
3f9d2c7a1e6b48d05c93a7f21e4d8b60
```

The value exists in canonical `main` history of `Saibar-Security/Attestation`:

```text
commit  f00dc0634c10dd55c3a4d0ec64084bc205b4d577
subject Point CI at the new deployment secret
file    .github/workflows/ci.yml
```

Exposed line:

```yaml
SECRET_KEY: "3f9d2c7a1e6b48d05c93a7f21e4d8b60"
```

Descendant commit `abb797712f1950c3fcc64f61dc4a0b990bc95136` replaced it with:

```yaml
SECRET_KEY: ${{ secrets.LARKSPUR_SECRET_KEY }}
```

Its commit body explains that inlining the secret meant every fork could read it.

## High-level overview

```text
first Drive ZIP
→ four QR quadrants
→ reconstructed QR
→ second Drive ZIP
→ valentine.png
→ row-major interleaved RGB LSB extraction
→ "Saibar-Security"
→ GitHub account and Attestation repository
→ canonical Git-history credential
→ pbctf{credential}
```

## Step-by-step procedure

### 1. Download the first archive

```powershell
$case = 'C:\Users\kingg\Documents\codex 2\shit-secrets-ctf'

Invoke-WebRequest `
  -Uri 'https://drive.usercontent.google.com/download?id=1JQl11pSQHCh7NA1IcJB7H043g489pyQU&export=download&confirm=t' `
  -OutFile "$case\dustbin"

Get-FileHash -Algorithm SHA256 -LiteralPath "$case\dustbin"
tar -tf "$case\dustbin"
tar -xf "$case\dustbin" -C $case
```

First-archive SHA-256:

```text
C1671A2FE9A1213489653CFE21F40ABC3D83DA7A8EF1BADFAF4FE74DEE7FEABA
```

Members:

```text
33b68a4a402e48cfaf334ddd879f54cd.png
ae8f1babf9f8433eb53b83fe0aa0bcda.png
547188aff27a47d5bf5829e9871ed84c.png
f577ef434f4c4bf4b289dab1cff3ff62.png
```

### 2. Reconstruct the QR

The white margins identify the quadrants:

| Fragment | Non-white bounding box | Position |
|---|---:|---|
| `33b68...png` | `(575,575,4096,4096)` | top-left |
| `ae8f...png` | `(0,575,3521,4096)` | top-right |
| `5471...png` | `(575,0,4096,3521)` | bottom-left |
| `f577...png` | `(0,0,3521,3521)` | bottom-right |

```python
from pathlib import Path
from PIL import Image

root = Path(r"C:\Users\kingg\Documents\codex 2\shit-secrets-ctf")

files = {
    "tl": "33b68a4a402e48cfaf334ddd879f54cd.png",
    "tr": "ae8f1babf9f8433eb53b83fe0aa0bcda.png",
    "bl": "547188aff27a47d5bf5829e9871ed84c.png",
    "br": "f577ef434f4c4bf4b289dab1cff3ff62.png",
}

parts = {k: Image.open(root / v).convert("RGB") for k, v in files.items()}
qr = Image.new("RGB", (8192, 8192), "white")
qr.paste(parts["tl"], (0, 0))
qr.paste(parts["tr"], (4096, 0))
qr.paste(parts["bl"], (0, 4096))
qr.paste(parts["br"], (4096, 4096))
qr.resize((2048, 2048), Image.Resampling.NEAREST).save(root / "reconstructed.png")
```

Decode:

```powershell
py -3.12 -m pip install pillow zxing-cpp
py -3.12 -c "import zxingcpp; from PIL import Image; print([x.text for x in zxingcpp.read_barcodes(Image.open(r'C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\reconstructed.png'))])"
```

Destination:

```text
https://drive.google.com/file/d/1SBZQvK6F5CUKaDCkKNvpvKMS6t7AfMBz/view?usp=sharing
```

### 3. Download the second archive

```powershell
Invoke-WebRequest `
  -Uri 'https://drive.usercontent.google.com/download?id=1SBZQvK6F5CUKaDCkKNvpvKMS6t7AfMBz&export=download&confirm=t' `
  -OutFile "$case\qr-target"

tar -tf "$case\qr-target"
tar -xf "$case\qr-target" -C $case
```

The only member is `valentine.png`, RGB `3840 × 2160`.

Hashes:

```text
second archive:
4DA3F2DDCD656D9C2E1A05AC932AACCC902DB4BFD3D1ECF05F14674A04C5BFD5

valentine.png:
522C91B53C4FCF7C67DFB789FC40E015A03E10D75EDE1533BE1A0B6EFDAFB4DB
```

PNG chunks and CRCs were valid. There was no metadata or trailing data after `IEND`.

### 4. Extract the RGB-LSB payload

Correct parameters:

- Row-major pixel order
- Interleaved `R,G,B` channels
- Least-significant bit from every channel
- MSB-first byte packing
- First four bytes are a big-endian payload length

```python
from PIL import Image
import numpy as np

path = r"C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\valentine.png"
pixels = np.asarray(Image.open(path).convert("RGB"), dtype=np.uint8)

bits = (pixels.reshape(-1, 3) & 1).reshape(-1)
raw = np.packbits(bits, bitorder="big").tobytes()

length = int.from_bytes(raw[:4], "big")
payload = raw[4:4 + length]

print(length)
print(payload.decode())
print(raw[:19].hex())
```

Expected:

```text
15
Saibar-Security
0000000f5361696261722d5365637572697479
```

### 5. Follow the locator into GitHub history

```powershell
gh api users/Saibar-Security
gh api users/Saibar-Security/repos --paginate `
  --jq '.[] | [.name,.html_url,.default_branch] | @tsv'

git clone https://github.com/Saibar-Security/Attestation.git "$case\Attestation"
Set-Location "$case\Attestation"

git log --all --graph --decorate --oneline --date-order
git show f00dc0634c10dd55c3a4d0ec64084bc205b4d577:.github/workflows/ci.yml
git show abb797712f1950c3fcc64f61dc4a0b990bc95136:.github/workflows/ci.yml
```

The earlier deployment secret was:

```text
1f8b4d0c6a2e97b35d4f0e8a1c6b9d2f
```

Commit `f00dc063...` changed it to:

```text
3f9d2c7a1e6b48d05c93a7f21e4d8b60
```

The latter is the intended exposed credential.

## Skills and workflows used

- `solve-challenge`
- `ctf-forensics`
- `ctf-osint`
- Browser-control workflow for limited dashboard inspection
- QR-fragment geometry reconstruction
- PNG structure and steganography sweep
- Public-identity OSINT
- Canonical and rewritten Git-history audit

## Tools and services used

- Google Drive
- PowerShell
- `Invoke-WebRequest`, `Get-FileHash`, `Format-Hex`
- ZIP/`tar`
- Python 3.12
- Pillow
- NumPy
- ZXing-C++
- OpenCV
- Git
- GitHub CLI and REST API
- `rg`
- Local image inspection

## Evidence and controls

- Both downloads matched independent redownloads byte-for-byte.
- QR decoding was reproduced with ZXing and OpenCV.
- Only one tested LSB ordering yielded a valid length prefix and coherent locator.
- The final secret is in canonical `main`, not only an unreachable Git object.
- A later canonical commit explicitly explains the exposure.
- The current supplied scoreboard lists the challenge as solved.

## Surviving files

```text
C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\dustbin
C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\reconstructed.png
C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\qr-target
C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\valentine.png
C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\audit_media.py
C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\bitplanes.png
C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\channel_diffs.png
C:\Users\kingg\Documents\codex 2\shit-secrets-ctf\Attestation
```

## Warning: rejected and decoy candidates

| Candidate | Disposition |
|---|---|
| `pbctf{r0t4t10n_r3c0rds_b13ed_thr0ugh_t00}` | Explicitly shown as incorrect |
| `pbctf{ch3cksums_l13d_4nd_th3_bundl3_t0ld_0n_th3m}` | Rejected checksum trap |
| `pbctf{n3v3r_c0mm1t_pr0d_s3cr3ts_2_g1t}` | Rejected backup-history material |
| `pbctf{r0t4t3d_4nd_r3v0k3d_l0ng_ag0}` | Superseded material |
| `pbctf{st4g1ng_1s_n0t_pr0duct10n}` | Staging decoy |
| `pbctf{f1xtur3_d4t4_1s_n0t_3v1d3nc3}` | Fixture decoy |
| `pbctf{d3m0_d4t4_1s_n0t_ev1d3nc3}` | Demo decoy |
