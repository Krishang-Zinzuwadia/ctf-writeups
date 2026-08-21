# Spiegel — The Grimoire of Equivalent Spells

## Challenge identity

- **Official scoreboard name:** Spiegel-The Grimoire of Equivalent Spells
- **Points:** 399
- **Category:** Cryptography
- **Status:** **CONFIRMED SOLVED**
- **Confirmed flag:** `pbctf{die_etiketten_waren_nie_zauber_me-jain-anurag}`
- **Diagram:** [`diagrams/spiegel-pipeline.excalidraw`](diagrams/spiegel-pipeline.excalidraw)

The supplied scoreboard independently confirms the name, category, value, and solved status.

## Original question

> *Der Spiegel zeigt nicht, was du bist. Er zeigt, was gleich aussieht.*
>
> Flamme's estate held one locked case: a mirror, a book, and a recording.
>
> The book is a grimoire of **equivalent spells** — pairs that come out the same against anything you can measure. Equivalence, she wrote in the margin, is a property of the observer, not of the spell.
>
> The **tablet** beside it was copied in an unfamiliar hand. The spells on it are genuine; the transcriber kept only their order, not their names.
>
> The **recording** is Flamme's own voice. She is not explaining anything — she is counting.
>
> Her warding golem still answers. It will bind a spell for you and say whether what you bound was the right one. Nothing more.
>
> https://drive.google.com/file/d/156vddlMbWelmLYLc0qOF8udcRCtlnRqQ/view?usp=sharing
>
> `nc 172.198.160.128 1342`

The only explicit user hint was:

> cryptography

## High-level overview

The challenge supplied three views of a finite Heisenberg/Weil representation:

- `spruchtafel.txt` contained twelve dense `169 × 169` projective Weyl operators over \(\mathbb F_{200201}\).
- `chant.wav` encoded twelve four-component labels as 48 base-13 tones.
- Pairwise operator commutators yielded 66 discrete-log exponents used to decrypt the grimoire appendix.

The appendix revealed a symplectic `4 × 4` coordinate frame. That frame was used to reconstruct the tablet’s exact raw Weyl convention, including scalar phases, and to recover the hidden observer basis.

The live service then sent eight words of 14 metaplectic gates (`C`, `L`, `F`) acting on 169-element vectors. Applying the exact finite Weil representation in the recovered observer basis passed all eight rounds and returned the flag.

## Artifact integrity and parameters

Archive SHA-256:

```text
97BD00D2155D2D3947F6C65D5D17BDF1FC3096466A33A4DE94543CA33B35CDE1
```

Extracted files:

```text
spruchtafel.txt
grimoire.txt
chant.wav
checksums.txt
```

Core parameters:

```text
p   = 13
n   = 2
ell = 200201
m   = 12
d   = 169
```

## Step-by-step procedure

### 1. Decode the tablet

`spruchtafel.txt` stores Base64-encoded zlib data:

```python
import base64
import numpy as np
import zlib

text = open("spruchtafel.txt").read()
raw = zlib.decompress(base64.b64decode(text.split("data:\n", 1)[1]))
matrices = np.frombuffer(raw, dtype="<u4").reshape(12, 169, 169)
```

Decompressed size:

```text
12 × 169 × 169 × 4 = 1,370,928 bytes
```

### 2. Decode the recording

`chant.wav` is mono 16-bit PCM at 8 kHz and lasts about 21.49 seconds. It consists of 48 pure-tone segments grouped as twelve labels of four tones.

Tone mapping:

\[
f = 500 + 15d\ \text{Hz}, \qquad d \in \{0,\ldots,12\}
\]

Recovered labels:

```text
(6,4,7,12)
(11,12,9,11)
(7,8,9,6)
(5,2,5,5)
(9,7,9,4)
(5,9,2,1)
(10,2,6,0)
(7,2,1,6)
(4,7,2,9)
(6,10,7,3)
(6,1,3,5)
(11,1,2,0)
```

Cloud speech transcription was unavailable because no `OPENAI_API_KEY` was configured. Offline waveform/pitch analysis was sufficient because the file contained tones rather than speech.

### 3. Compute projective commutators

For every pair \(i<j\):

\[
A_iA_j=s_{ij}A_jA_i
\]

Select a nonzero corresponding matrix entry:

\[
s_{ij}=(A_iA_j)_{rc}(A_jA_i)_{rc}^{-1}\pmod{200201}
\]

Verify the ratio against every entry, not just the selected one.

The scalars are 13th roots of unity. Using `5144` as the discrete-log base produced 66 exponents.

### 4. Derive the appendix key

Serialize each exponent as unsigned two-byte big-endian data in pair-row order:

```text
(0,1), (0,2), ..., (0,11), (1,2), ...
```

Prefix:

```text
spiegel/v1/frame-key|p=13|n=2|ell=200201|m=12|
```

Compute:

\[
K=\operatorname{SHA256}(\text{prefix}\parallel k_{01}\parallel k_{02}\parallel\cdots)
\]

Recovered key:

```text
705ca779de869efc9b2da5e0ec105d1e1610fb66715641c7bd7660948b943e3f
```

### 5. Decrypt the rune appendix

Map the rune names to nibbles:

```text
0 alt      1 brom     2 dain     3 eisen
4 fern     5 glut     6 hain     7 irr
8 kalt     9 lind     a morg     b nebel
c ost      d riss     e still    f turm
```

Generate the XOR veil:

\[
V_z=\operatorname{SHA256}(K\parallel z_{\text{BE32}})
\]

The plaintext begins with the control header:

```text
SPGL1
```

Recovered frame:

```text
[[7,3,5,7],
 [1,4,5,8],
 [5,3,11,3],
 [4,10,8,7]]
```

Control:

\[
F^TJF=J\pmod{13}
\]

### 6. Resolve the canonical Weyl root

For labels \(v=(q,p)\) and \(w=(q',p')\):

\[
\langle v,w\rangle=q\cdot p'-p\cdot q'
\]

The observed commutator exponent using base `5144` was:

\[
k_{ij}=8\langle v_i,v_j\rangle\pmod{13}
\]

Therefore:

\[
\omega=5144^8\bmod 200201=77581
\]

### 7. Recover the exact tablet convention

The critical correction was that the decrypted frame described the tablet’s coordinate system; it was not the live service’s `F` gate.

For transformed label \(u=Fv=(q,p)\), the exact tablet representative uses the raw \(Z^pX^q\) phase:

\[
Q(u)=-\frac12 q\cdot p\pmod{13}
\]

Four independent tablet labels were used as a basis. The phase correction made all twelve reconstructed operators equal the supplied matrices exactly—not merely proportional up to a scalar.

### 8. Build the hidden observer basis

Recover the four unit operators:

```text
X1 = W(1,0,0,0)
X2 = W(0,1,0,0)
Z1 = W(0,0,1,0)
Z2 = W(0,0,0,1)
```

Project onto the common eigenvalue-1 space of `Z1` and `Z2`, then generate:

\[
X_1^aX_2^b|0,0\rangle,\qquad a,b\in\mathbb F_{13}
\]

Stack the 169 basis vectors into observer matrix \(S\), invert it modulo `200201`, and verify:

\[
S^{-1}S=I
\]

### 9. Implement the live gate representation

Service banner:

```text
SPIEGEL/1 PARAMS p=13 n=2 ell=200201 m=12 d=169 rounds=8 word_len=14 deadline=600
commands: PARAMS | BEGIN | ANS <base64> | QUIT
```

For each round:

1. Decode the 169 little-endian `uint32` field elements.
2. Convert to canonical coordinates: \(x=S^{-1}y\).
3. Apply the 14 gates.
4. Convert back: \(y'=Sx'\).
5. Pack 169 little-endian `uint32` values.
6. Base64-encode and send `ANS <base64>`.

Successful convention:

```text
coordinate indexing = big-endian
word order          = reversed
C chirp sign        = -1
quadratic factor    = 1/2
L action            = A^-1
L scalar            = Legendre(det(A))
F exponent sign     = -1
F normalization     = 1/13
```

Gate formulas:

\[
(C_Bf)(x)=\omega^{-\frac12x^TBx}f(x)
\]

\[
(L_Af)(x)=\chi(\det A)\,f(A^{-1}x)
\]

\[
(Ff)(x)=\frac1{13}\sum_y\omega^{-x\cdot y}f(y)
\]

### 10. Reproduce with the surviving solver

From `C:\Users\kingg\Documents\codex 2`:

```powershell
@'
from spiegel.hidden_solver import TabletRepresentation, build_observer, attempt, canonical

tablet = TabletRepresentation()
observer, observer_inverse = build_observer(tablet, "raw")

variant = canonical.Variant(
    False,
    True,
    -1,
    True,
    "Ai",
    True,
    -1,
    "invp",
)

print(attempt(observer, observer_inverse, variant, complete=True))
'@ | uv run --with numpy python -
```

Expected final progression:

```text
OK 1
OK 2
OK 3
OK 4
OK 5
OK 6
OK 7
OK 8
FLAG pbctf{die_etiketten_waren_nie_zauber_me-jain-anurag}
```

## Skills and workflows used

- `solve-challenge`
- `ctf-crypto`
- `transcribe` workflow; cloud path unavailable, replaced with offline tone analysis

## Tools, libraries, and services used

- Python 3
- NumPy
- `uv run --with numpy`
- `zlib`, `base64`, `hashlib`, `struct`, `socket`
- Offline waveform/FFT-style pitch analysis
- PowerShell
- Google Drive
- Netcat/ncat and Python sockets
- Live PBCTF oracle
- Web search and public-source searches

## Evidence and controls

- Archive and extracted-file hashes were recorded.
- Tablet dimensions exactly matched the declared parameters.
- Every commutator ratio was verified entry-wise.
- The appendix decrypted to `SPGL1`.
- The frame passed the symplectic identity.
- Audio labels reproduced the pairwise symplectic exponents.
- All twelve tablet matrices matched reconstructed raw Weyl operators exactly.
- Observer inversion passed modulo `200201`.
- One successful round was explicitly rejected as insufficient.
- The final normalization passed all eight live rounds.
- The remote service returned the flag.

## Surviving files

```text
C:\Users\kingg\Documents\codex 2\spiegel\solve_spiegel.py
C:\Users\kingg\Documents\codex 2\spiegel\hidden_solver.py
C:\Users\kingg\Documents\codex 2\spiegel\extracted\grimoire.txt
C:\Users\kingg\Documents\codex 2\spiegel\extracted\spruchtafel.txt
C:\Users\kingg\Documents\codex 2\spiegel\extracted\chant.wav
C:\Users\kingg\Documents\codex 2\spiegel\extracted\checksums.txt
C:\Users\kingg\Documents\codex 2\spiegel\download
```

## Rejected hypotheses and non-flags

- `5144` is the appendix discrete-log base.
- `77581` is the canonical Weyl root.
- Treating the decrypted frame as the live `F` gate was wrong.
- A one-round success was not accepted as proof.
- No rejected flag strings were produced.
