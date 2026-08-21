# Mimic
> Reconstructed from Codex task [019f9c71-cb1d-77f1-ba62-522af34abcd8](thread://019f9c71-cb1d-77f1-ba62-522af34abcd8). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge statement and supplied material

**Name:** Mimic  
**Points:** 350  
**Author:** ravenspar

> We intercepted the mimic's chatter — hundreds of frames, one broken cipher, and a great many faces that look almost right. Find the one true face.

- Original handout: [Google Drive](https://drive.google.com/file/d/1UoVLIvL3jdm0VpgfdEo8hNGncb0MBc47/view?usp=sharing)
- User-supplied local ZIP: `C:\Users\kingg\Desktop\codex2\mimic.zip`
- ZIP size: 24,186 bytes
- SHA-256: `4553CD0EE7255F2437E6B15149562DCE4E3D027E309B04C607EC54A43CD7E96B`

The handout contains `challenge.txt`, `spec.md`, `sbox.txt`, `intercepts.txt`, and `README.txt`. It describes an AES-shaped 128-bit, 10-round block cipher, a custom S-box, a deliberately simple round-key schedule, 300 encrypted frames, and a Fletcher-16 “seal” for the intended payload text.

## Category and status

- **Category:** Cryptography with a forensics/decoy-validation layer.
- **Status:** **Unresolved.**
- **Confirmed flag:** **None.**
- **Reason:** every flag reported to the live checker was rejected. The strongest artifact-internal candidate has a perfect same-frame transform and checksum, but that does not override the checker rejection.

## High-level solution overview

The custom S-box is affine over bits, which makes the full cipher affine in the 128 key bits despite looking AES-like. The known plaintext/ciphertext pair therefore gives a rank-128 linear system and uniquely recovers the channel key. Decrypting all frames yields 44 obvious flag-shaped “faces,” exactly as the story warns. None of those raw faces has its own frame’s Fletcher-16 seal.

A second pass found two competing structures:

1. Some face strings can be changed in two positions to collide with a frame seal. This produced a linguistically plausible `faux` candidate, but it was rejected.
2. Frame `0003` contains high-byte data that, after subtracting the repeating ASCII word `mirror`, becomes a clean flag-shaped candidate whose Fletcher-16 is exactly the same frame’s `f29f` seal. The local ZIP and a fresh Drive download were byte-identical, so this is genuine handout behavior, not file corruption. The live checker still rejected the string.

The defensible conclusion is that the distributed artifact and active checker were out of sync, or an additional unrepresented transformation was intended. No accepted flag was found.

## Detailed procedure

1. Validate the cipher implementation against the supplied known-key vector:

   ```text
   key        = 2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b
   plaintext  = 30313233343536373839616263646566
   ciphertext = be1ad07324596b544488b3c0d47007c5
   ```

2. Implement the cipher exactly as specified:

   - 16-byte, column-major AES state.
   - `SubBytes` via the supplied table.
   - AES `ShiftRows`.
   - AES MixColumns over GF(2^8) with polynomial `0x11B`.
   - Ten rounds.
   - Round-key byte:

     ```text
     rk[r][i] = key[i] XOR ((31*r + 7*i) mod 256)
     ```

3. Observe that the supplied S-box is affine and the rest of the cipher operations are affine/linear. Compute:

   ```text
   E(P, K) = E(P, 0) XOR L(K)
   ```

   Build the 128 columns of `L` by encrypting the known plaintext under one-hot keys. Gaussian elimination over GF(2) gives full rank 128.

4. Recover and verify the unique channel key:

   ```text
   ASCII: MimicV3SecretKy!
   HEX:   4d696d696356335365637265744b7921
   ```

   Verification pair:

   ```text
   known plaintext  = 6b6e6f776e2d706c61696e7465787421
   known ciphertext = 1b834f4080555f2780cfbadcaf65cd48
   ```

5. Decrypt all 300 three-block ECB frames and re-encrypt each plaintext as an integrity check. Extract `pbctf{...}`-shaped substrings.

6. Calculate each candidate’s Fletcher-16:

   ```python
   s1 = 0
   s2 = 0
   for b in payload:
       s1 = (s1 + b) % 255
       s2 = (s2 + s1) % 255
   seal = (s2 << 8) | s1
   ```

   Findings:

   - Frames: `300`
   - Obvious flag faces: `44`
   - Direct raw face → same-frame seal matches: `0`

7. Exhaustively solve the two-character Fletcher repair equations for each face. There were 222 printable same-frame repair possibilities. The best language-ranked repair changed the raw frame-0186 face into:

   ```text
   pbctf{0n3_true_faux_am0ng_th3_m1m1cs}
   ```

   It matched captured seal `3529`, but the checker rejected it.

8. Scan decrypted payloads for secondary transforms using keywords from the handout. Subtracting repeating `mirror` from frame `0003` yields:

   ```text
   pbctf{0n3_tru3_fac3_am0ng_th3_m1m1cs}
   ```

   Its Fletcher-16 is exactly frame `0003`’s seal `f29f`. This is the strongest internal handout evidence, but the live checker rejected it.

9. Compare the supplied ZIP with a fresh Drive download by SHA-256. They matched, ruling out a stale local copy relative to the linked file at that time.

10. Run wider structural scans over XOR, addition, subtraction, affine byte maps, keyword transforms, source/timestamp/seal-derived masks, substrings, and two-character checksum repairs. No alternative yielded a checker-confirmed flag.

## Important commands and algorithms

```powershell
Get-FileHash -Algorithm SHA256 -LiteralPath 'C:\Users\kingg\Desktop\codex2\mimic.zip'
python 'C:\Users\kingg\Documents\codex 2\mimic_fresh\fresh_solve.py'
python 'C:\Users\kingg\Documents\codex 2\mimic_fresh\keyword_transform_scan.py'
python 'C:\Users\kingg\Documents\codex 2\mimic_fresh\structure_scan.py'
```

Core algorithms:

- GF(2^8) AES-style state transformations.
- Affine-cipher key recovery as a 128×128 GF(2) linear system.
- Fletcher-16 modulo 255.
- Closed-form two-position checksum repair.
- Repeating-key byte subtraction with key `mirror`.

The current machine’s `fresh_solve.py` language-ranking dependency fails to import its compiled `regex._regex` extension, so reproducing the ranking requires reinstalling compatible `wordfreq`/`regex` wheels. The core cryptographic solver and recorded results remain available.

## Evidence and validation

- The cipher matches the handout test vector.
- Recovered key has full GF(2) rank 128 and reproduces the known channel ciphertext exactly.
- All decrypted frames re-encrypt to their original ciphertext.
- Frame `0003` + repeating subtraction key `mirror` produces a clean payload with same-frame Fletcher seal `f29f`.
- Local and Drive ZIPs matched.
- **Counter-evidence that controls final status:** the active checker rejected the resulting token and all other reported tokens.

## Rejected candidates and decoys

All reported candidates remain rejected:

- `pbctf{0n3_tru3_fac3_am0ng_th3_m1m1cs}` — strong frame-0003 transform/seal match, explicitly rejected.
- `pbctf{0n3_true_faux_am0ng_th3_m1m1cs}` — two-byte repair matching seal `3529`, explicitly rejected.
- `pbctf{0n3_true_fac3_am0ng_th3_m1m1cs}` — plausible face/normalization variant, rejected.
- `pbctf{0ne_true_f4ce_4mong_7h3_m1m1c5}` — checksum-collision/leet candidate reported later, rejected according to the complete conversation record.

The other 40+ decrypted strings are deliberate “faces” and include Rickroll fragments, “hidden mirror/shadow/mask” phrases, and explicit trap language. They are not flags merely because they match the wrapper.

## Codex skills used

- `solve-challenge`
- `ctf-crypto`
- `ctf-forensics`
- `browser:control-in-app-browser` for portal/checker inspection

## Tools, utilities, libraries, and services

- Python 3 custom cipher, GF(2), checksum, and scan scripts.
- PowerShell, `Get-FileHash`, `Get-Content`, `Select-String`, archive utilities.
- Google Drive download endpoints.
- PBCTF/CTFd web portal and its live flag checker.
- In-app Chromium/browser automation for authenticated checker inspection.
- `wordfreq` and `regex` for optional language ranking.

## Local artifacts

- `C:\Users\kingg\Desktop\codex2\mimic.zip`
- `C:\Users\kingg\Documents\codex 2\mimic_work\mimic_linked.zip`
- `C:\Users\kingg\Documents\codex 2\mimic_work\solve_mimic.py`
- `C:\Users\kingg\Documents\codex 2\mimic_fresh\fresh_solve.py`
- `C:\Users\kingg\Documents\codex 2\mimic_fresh\keyword_transform_scan.py`
- `C:\Users\kingg\Documents\codex 2\mimic_fresh\structure_scan.py`
- `C:\Users\kingg\Documents\codex 2\mimic_fresh\challenge.txt`
- `C:\Users\kingg\Documents\codex 2\mimic_fresh\spec.md`
- `C:\Users\kingg\Documents\codex 2\mimic_fresh\sbox.txt`
- `C:\Users\kingg\Documents\codex 2\mimic_fresh\intercepts.txt`

## Diagram assessment

**Yes, a diagram materially helps.** Suggested nodes and edges:

```text
ZIP → parse cipher/S-box → GF(2) key recovery → key MimicV3SecretKy!
key → decrypt 300 frames → 44 flag faces → Fletcher checks/repairs
decrypted frame 0003 → subtract "mirror" → f29f-matching candidate
candidate → live checker → rejected → unresolved artifact/checker mismatch
```

---

## Final High-Level Overview

The affine S-box reduced key recovery to a full-rank GF(2) system, revealing the cipher key and all 300 plaintext frames. Several same-frame checksum candidates were reconstructed, but every submitted token was rejected; the handout and checker therefore remain inconsistent and no flag is confirmed.
