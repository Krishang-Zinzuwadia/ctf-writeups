# Backfill / Interrupted Registry Migration
> Reconstructed from Codex task [019f9cbe-3232-7381-84d0-42ac6e76a17e](thread://019f9cbe-3232-7381-84d0-42ac6e76a17e). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge record

- **Exact in-archive challenge name:** `Backfill`
- **User-facing description:** interrupted registry migration
- **Link:** <https://drive.google.com/drive/folders/1Kt1AVGBc_iG3FVTaMldx2VgB6HiNRQey?usp=sharing>
- **Category:** Forensics and Cryptography; specifically OCI-registry artifact recovery plus finite-field matrix reconstruction
- **Status:** **Confirmed solved**
- **Confirmed flag:** `pbctf{th3_c0s3ts_w3r3_th3_c4rg0_and_th3_h0l3s_w3r3_th3_k3y}`

The user supplied:

> A registry migration was interrupted.
>
> The backup that survived appears complete enough to be useful, yet incomplete enough to be suspicious.
>
> Everything you need is in the archive.
>
> [Way to registry](https://drive.google.com/drive/folders/1Kt1AVGBc_iG3FVTaMldx2VgB6HiNRQey?usp=sharing)

The word “registry” referred to an OCI image registry rather than the Windows Registry.

## High-level solution

The backup held an OCI index containing 512 numbered manifests and a sealed flag. Its matrix records came in pairs: record `2i+1` was the additive inverse of record `2i` modulo `268435361`. Thirty-two complete records revealed a 16-element XOR subgroup. Partitioning the remaining records into its 16 cosets reduced reconstruction to overdetermined 16-variable linear systems over the finite field. Once all matrices were recovered and canonically serialized, their SHA-256 digest derived encryption and MAC keys. The supplied HMAC verified before AES-like SHA-256 counter-stream decryption produced the flag.

## Acquisition and integrity

The Drive folder contained an attachment folder and these relevant IDs:

- Attachment folder: `1fahdeWLHgFDYLDzvooDmFa4GbS5NsHKD`
- `backfill.tar.gz`: `1AB2mmgdRzplsx6udMNjHJ0qNTxcNTE0B`
- `README.md`: `1S1zdFgCJtBluSAkgXvJzkIgszaopYT8u`
- `checksums.txt`: `14J02EPoFG13du-y_TH4WyTfvjZ9ECs3E`

The downloaded archive had:

```text
Size:   773074 bytes
SHA256: 4e06e26b69a5de54027c5c8b87890c12a0f66a3b904d8e65212c06fdc289bc89
```

That digest exactly matched the supplied checksum file.

## Archive structure

Inspection found:

- 512 OCI manifests/tags named `backfill:0000` through `backfill:0511`;
- 1,075 referenced blobs;
- 51 unique layers:
  - 3 common base layers,
  - 16 cache layers,
  - 16 shard layers,
  - 16 unit layers;
- a fingerprint schema with matrix order 16 and modulus `268435361`;
- 512 records:
  - 384 records with 160 observed cells,
  - 96 empty records,
  - 32 complete 16×16 records;
- `flag.sealed.json`.

## Detailed procedure and reconstruction

1. **Normalize the paired records.**

   For every `i`, tags `(2i, 2i+1)` represented `M` and `-M mod P`. This reduced the structural problem to 256 even-indexed matrices while also providing a consistency check.

2. **Identify the complete subgroup.**

   The 16 independent complete even matrices were indexed:

   ```text
   0, 2, 9, 11, 36, 38, 45, 47,
   144, 146, 153, 155, 180, 182, 189, 191
   ```

   Their index set is closed under bitwise XOR, revealing a 16-element subgroup rather than an arbitrary selection of complete records.

3. **Partition all indices into cosets.**

   The 256 even matrices were partitioned into 16 cosets of that subgroup. Known subgroup matrices could then act as left multipliers on an unknown coset representative.

4. **Recover incomplete matrices over the field.**

   For each incomplete coset and each matrix column, the observed cells generated an overdetermined system with 16 unknowns over:

   ```text
   GF(268435361)
   ```

   Modular Gaussian elimination solved the systems. Scalar ratios across independent observed cells were then checked to reject inconsistent projective choices.

   This recovered:

   - 208 nonempty matrices exactly;
   - 48 empty matrices projectively.

5. **Fix the remaining signs.**

   Canonical generator matrices were associated with indices:

   ```text
   1, 2, 4, 8, 16, 32, 64, 128
   ```

   Ordered generator products reproduced the known matrices. All 208 known section signs were zero, and the structurally consistent completion assigned zero to the remaining 48 signs. Odd-indexed records were then restored as additive inverses.

6. **Serialize the complete matrices.**

   Complete matrices were sorted by target digest. Every entry was serialized row-major as an unsigned four-byte big-endian integer. Hashing the resulting canonical stream yielded:

   ```text
   key = 97da830d87df5305021327b47d4a56d1a0306ccb0c8c1e3d6a5cef07bb922692
   ```

7. **Derive cryptographic keys and verify before decrypting.**

   The archive specified:

   ```text
   key   = SHA256(canonical complete-matrix serialization)
   k_enc = SHA256("backfill/enc" || key)
   k_mac = SHA256("backfill/mac" || key)
   ```

   The sealed object contained:

   ```text
   ciphertext = 126afa4aca6d8627024ef138c4f9be54ed5b5cdb13d9dbf6f9221d4d57dd38b3a34ff96d731fb47e9ed9e42836d3219612ab7711d811aa04be93d8
   nonce      = 1fe087b9949d5d4c94c825de285e605b
   tag        = 8699361f20167f33c8aba6c0590558ee9fdfbddcd6f1cce051ded466f9c321c1
   scheme     = sha256-ctr-then-hmac-sha256
   ```

   The stream was generated in blocks as:

   ```text
   SHA256(k_enc || nonce || counter_be32)
   ```

   Before decrypting, the solver checked:

   ```text
   HMAC-SHA256(k_mac, nonce || ciphertext) == tag
   ```

   The result was `MAC: verified`. XORing the ciphertext with the counter stream then produced the exact flag.

## Important command

The surviving solver can reproduce the reconstruction and validation:

```powershell
python .\solve_registry.py
```

Expected decisive output includes the 16-member complete subgroup, `exact 208`, `projective 48`, the key above, `MAC: verified`, and:

```text
pbctf{th3_c0s3ts_w3r3_th3_c4rg0_and_th3_h0l3s_w3r3_th3_k3y}
```

## Tools, libraries, services, and Codex skills

- **Codex skills used:** `solve-challenge`, `ctf-forensics`, `browser:control-in-app-browser`.
- **Tools/services:** Google Drive, in-app browser, PowerShell, `Invoke-WebRequest`, `tar`, Python.
- **Python modules/algorithms:** `json`, `hashlib`, `hmac`, `tarfile`, `collections`, modular Gaussian elimination, finite-field arithmetic, canonical binary serialization.

## Validation evidence

This flag is cryptographically confirmed:

1. the downloaded archive matched the organizer-supplied SHA-256;
2. the reconstructed matrix serialization derived a deterministic key;
3. that key made the supplied HMAC verify exactly;
4. the authenticated ciphertext decrypted to a syntactically complete `pbctf{...}` flag.

The HMAC check is important: it distinguishes the result from a plausible-looking decryption produced with a wrong reconstruction.

## Rejected approaches and why

- **Windows Registry interpretation:** rejected after the archive proved to be an OCI registry.
- **Simple layer-sum model:** did not explain the record structure.
- **Ordinary linear span of the 32 complete records:** raw matrices had full rank while the relevant structure reduced to rank 16; a naive span did not reconstruct the missing records.
- **LCG, determinant, and symplectic-property theories:** failed to provide a consistent global reconstruction.
- **Tar metadata and public write-up searches:** yielded no flag or useful hidden channel.

## Local artifacts

| Artifact | Path |
|---|---|
| Final solver | `C:\Users\kingg\Documents\codex 2\solve_registry.py` |
| Downloaded archive | `C:\Users\kingg\AppData\Local\Temp\mnemo-registry\backfill.tar.gz` |
| README | `C:\Users\kingg\AppData\Local\Temp\mnemo-registry\README.md` |
| Supplied checksums | `C:\Users\kingg\AppData\Local\Temp\mnemo-registry\checksums.txt` |
| Extracted fingerprints | `C:\Users\kingg\AppData\Local\Temp\mnemo-registry\extracted\backfill\fingerprints.json` |
| Sealed flag | `C:\Users\kingg\AppData\Local\Temp\mnemo-registry\extracted\backfill\flag.sealed.json` |
| OCI index | `C:\Users\kingg\AppData\Local\Temp\mnemo-registry\extracted\backfill\registry\index.json` |
| OCI blobs | `C:\Users\kingg\AppData\Local\Temp\mnemo-registry\extracted\backfill\registry\blobs\sha256\` |

## Diagram assessment

**A diagram would materially help.** Concise graph specification:

- Nodes: `OCI tags`, `M/-M pair normalization`, `complete XOR subgroup`, `16 cosets`, `GF(P) linear systems`, `512 reconstructed matrices`, `canonical serialization`, `master key`, `HMAC check`, `counter-stream decrypt`, `flag`.
- Edges: `tags -> pair normalization -> subgroup -> cosets -> linear systems -> matrices -> serialization -> master key`; then branch `master key -> HMAC check` and `master key -> counter-stream decrypt`, with `HMAC check -> authenticated decrypt -> flag`.

---

## Excalidraw diagram

Editable diagram: [Backfill / Interrupted Registry Migration flow](diagrams/backfill-recovery.excalidraw)

## Final High-Level Overview

The OCI records were normalized into matrix pairs, reconstructed through an XOR subgroup and finite-field linear systems, canonically serialized, and used to derive authenticated decryption keys. The matching HMAC makes the recovered flag cryptographically conclusive.
