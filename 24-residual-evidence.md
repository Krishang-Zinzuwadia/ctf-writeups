# Residual Evidence
> Reconstructed from Codex task [019f9cbe-3232-7381-84d0-42ac6e76a17e](thread://019f9cbe-3232-7381-84d0-42ac6e76a17e). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge record

- **Exact name:** `Residual Evidence`
- **Link:** <https://drive.google.com/drive/folders/1OzuNy3Viq_rcBBA2K6N-n_nUh-oXn_97?usp=sharing>
- **Drive file ID:** `1DnEp2GHuFAZaQePsS3_SiW2JQxpm2HKH`
- **Category:** Forensics
- **Value discrepancy:** the user pasted 300 points; the later live CTFd API recorded challenge ID 26 at 400 points
- **Status:** **Unresolved**
- **Confirmed flag:** **None**
- **Platform state:** `solved_by_me: false`

The user supplied:

> **Residual Evidence**
> **300**
>  0  0
> A routine quarterly security review was archived after an internal records audit. Everything appears normal at first glance, but the recovered copy does not fully match the organization’s records. Find what was left behind.
>
> [Download the audit](https://drive.google.com/drive/folders/1OzuNy3Viq_rcBBA2K6N-n_nUh-oXn_97?usp=sharing)

## High-level overview

The PDF was structurally inspected, rendered, and unpacked. Its metadata and embedded notes contained two deliberate fake flags, while a zero-width stream, a controlled-vocabulary stream, and a sealed 54-byte blob formed the real unresolved core. Extensive authenticated-encryption and stream-combination searches produced no verified plaintext, so the write-up records the evidence without inventing a flag.

## Detailed procedure: acquire the file

The Drive folder contained `PaperTrail.pdf`:

```text
Size:   9881 bytes
SHA256: E459B375693748ADAE1931AF701828164D231F3EAE0CFDADBA9296DF17F1411A
```

The visible one-page document was a NORTHSTAR SYSTEMS Change Advisory Review:

- review ID `CAR-2026-08`;
- date `22 Aug 2026`;
- version `1.0`;
- an Appendix A note that records had been retained.

## PDF structural findings

The PDF had:

- one cross-reference table and one `%%EOF`;
- 14 numbered objects;
- no `/Prev` chain;
- no orphaned incremental-update objects;
- no bytes after the terminal EOF.

That ruled out the most obvious incremental-update and post-EOF hiding techniques. The meaningful residue was instead in metadata and embedded attachments.

## Proven decoy 1: metadata

The PDF Keywords field contained:

```text
cGJjdGZ7bTN0NGQ0dDRfbTFzZDFyM2N0MTBufQ==
```

Base64 decoding produced:

```text
pbctf{m3t4d4t4_m1sd1r3ct10n}
```

This is **not a confirmed flag**. The document's recovered material explicitly characterized metadata as misdirection and directed attention toward deeper attachments. It is a deliberate, syntactically valid decoy.

## Embedded attachments

Two embedded files were extracted:

| Attachment | Object | Size | MD5 |
|---|---:|---:|---|
| `IC-Log-Archive.txt` | 11 | 4,926 bytes | `810d7f5a5007dc201380d3cdcf4d067b` |
| `notes_FINAL_v2.txt` | 14 | 1,034 bytes | `edf32da5e258688872417abf463e9440` |

## Proven decoy 2: ROT13 note

The notes contained:

```text
copgs{e0g13_e3q_u3ee1at}
```

ROT13 produced:

```text
pbctf{r0t13_r3d_h3rr1ng}
```

This is also **not a confirmed flag**. Its plaintext literally labels itself a red herring, so it was correctly rejected.

## Remaining hidden channels

### Sealed base64 blob

Three lines concatenated to:

```text
9UCR4p+KCNqWU1cwCdv+Z85kx76lGYjzYiw1qeBQuS/pAE/TMjJuOhp9bGJE6tp1ePhCGKZp
```

Base64 decoding yielded 54 bytes:

```text
f54091e29f8a08da9653573009dbfe67ce64c7bea51988f3622c35a9e050b92fe9004fd332326e3a1a7d6c6244eada7578f84218a669
```

### Controlled word stream

The word soup had 88 tokens drawn from 15 words:

```text
synergy envelope matrix cadence vector payload handshake manifest
baseline artifact residual checksum cipher buffer offset
```

Assigning nibbles by first appearance produced 44 bytes:

```text
0123345167890083379071517a3b2cd0cc41ae43ce88a8de92c01b86a872d5969318053c8d3d8a2b4ee35216
```

Mapping the natural word lengths 6–9 into two-bit values produced another 22-byte stream:

```text
6146ea5969a6a60102810aa081aba81b9a5484a2014b
```

### Zero-width stream

`IC-Log-Archive.txt` contained 264 zero-width characters:

- U+200C: 142 occurrences
- U+200B: 122 occurrences

Using U+200B as 0 and U+200C as 1 gave 33 bytes:

```text
e5ae4b40baaf9db9604fab17617029867cfdfe2d6063dfaa62ef7052672b779598
```

Bit-reversing each byte gave:

```text
a775d2025df5b99d06f2d5e8860e94613ebf7fb406c6fb5546f70e4ae6d4eea919
```

Complement and complement-plus-bit-reversal variants were also retained as candidate materials.

## Cryptographic search performed

The extracted streams were tested systematically as keys, nonces, associated data, ciphertext layouts, and KDF inputs. The search included:

- AES-GCM;
- ChaCha20-Poly1305 and XChaCha20-Poly1305;
- AES-EAX, OCB, SIV, and CCM;
- AES-CTR, OFB, CFB, CBC, and ECB;
- RC4, Salsa20, and ChaCha20;
- PyNaCl SecretBox with embedded-nonce variants;
- XOR and repeating-XOR constructions;
- raw SHA-family hashes and concatenated KDF candidates;
- PBKDF2 candidates;
- embedded nonce/tag splits at plausible boundaries;
- all tested key/nonce/AAD role assignments;
- true base-15 packing of the 15-word alphabet;
- 3:2 combinations of the hidden streams;
- parity removal and word first/last/length property channels;
- line and matrix permutations;
- ROT13-before/after-base64 transformations;
- cross-stream XOR;
- minimum-Hamming-distance comparisons between candidate streams.

One background exhaustive run recorded:

```text
324 keys
192 candidate materials
746496 attempts
no authentication success
```

No tested combination authenticated, decrypted to a verifiable organizer message, or yielded a non-decoy flag.

## Why the status remains unresolved

The investigation proved that the PDF contains more than the two obvious bait strings, and it recovered several high-entropy hidden streams. It did **not** establish the intended transform that binds those streams to the 54-byte sealed blob. Neither a successful authentication tag nor a challenge-service confirmation exists.

The only two `pbctf{...}` strings found are explicitly self-invalidating decoys:

```text
pbctf{m3t4d4t4_m1sd1r3ct10n}   # metadata misdirection
pbctf{r0t13_r3d_h3rr1ng}       # says red herring
```

Neither may be promoted as the answer.

The two locked CTFd hints, IDs 12 and 13 with costs 10 and 50, were not purchased and therefore added no evidence.

## Tools, libraries, services, and Codex skills

- **Codex skills used:** `solve-challenge`, `ctf-forensics`, `pdf`, `computer-use:computer-use`.
- **Tools/services:** Google Drive, CTFd, browser/Computer Use, PowerShell, `curl`, `pdftoppm`, Node.js, Python.
- **PDF tooling:** `pypdf`, `pdfplumber`, PDF rendering and visual inspection.
- **Python/crypto tooling:** `re`, `zlib`, `base64`, `hashlib`, `collections`, `itertools`, NumPy, PyCryptodome, PyNaCl.
- **Node tooling:** built-in `crypto`.
- **External research:** web search for public write-ups; none supplied a confirmed solution.

## Local artifacts

| Artifact | Path |
|---|---|
| Original PDF | `C:\Users\kingg\Documents\codex 2\PaperTrail.pdf` |
| Saved Drive page | `C:\Users\kingg\Documents\codex 2\residual_drive.html` |
| Rendered PDF page | `C:\Users\kingg\Documents\codex 2\tmp\pdfs\PaperTrail-1.png` |
| Initial probe | `C:\Users\kingg\Documents\codex 2\residual_probe.py` |
| Stream combiner | `C:\Users\kingg\Documents\codex 2\residual_combine.py` |
| AES brute-force script | `C:\Users\kingg\Documents\codex 2\residual_aes_brute.js` |
| Embedded-layout search | `C:\Users\kingg\Documents\codex 2\residual_embedded.js` |
| Embedded search output | `C:\Users\kingg\Documents\codex 2\residual_embedded.out` |
| Embedded search errors | `C:\Users\kingg\Documents\codex 2\residual_embedded.err` |
| Role tests | `C:\Users\kingg\Documents\codex 2\residual_roles.py` |
| Quick role tests | `C:\Users\kingg\Documents\codex 2\residual_roles_quick.py` |
| AAD tests | `C:\Users\kingg\Documents\codex 2\residual_aad.py` |
| Base-15 tests | `C:\Users\kingg\Documents\codex 2\residual_base15.py` |
| General role search | `C:\Users\kingg\Documents\codex 2\residual_roles_general.py` |
| XOR search | `C:\Users\kingg\Documents\codex 2\residual_xor.py` |
| SecretBox search | `C:\Users\kingg\Documents\codex 2\residual_secretbox_embedded.py` |
| Distance analysis | `C:\Users\kingg\Documents\codex 2\residual_distance.py` |
| Local PyCryptodome install | `C:\Users\kingg\Documents\codex 2\tmp\pycrypto\` |
| Local PyNaCl install | `C:\Users\kingg\Documents\codex 2\tmp\pynacl\` |

## Diagram assessment

**A diagram would materially help.** Concise graph specification:

- Root node: `PaperTrail.pdf`.
- First branch: `metadata -> base64 -> metadata-misdirection decoy`.
- Second branch: `embedded files`.
- Under `embedded files`: `notes -> ROT13 -> red-herring decoy`; and `IC log -> zero-width bits`, `word soup -> nibble/length streams`, `three base64 lines -> 54-byte sealed blob`.
- Merge `zero-width bits`, `word streams`, and `sealed blob` into `tested KDF/key/nonce/AAD layouts`.
- Final node: `no authenticated decrypt`; color both decoy nodes red and the final status amber.

---

## Final High-Level Overview

The PDF yielded two embedded files, two self-invalidating decoy flags, several covert bit/word streams, and a sealed blob. No tested key, nonce, AAD, or container assignment authenticated, so the challenge remains unresolved and neither decoy is reported as the answer.
