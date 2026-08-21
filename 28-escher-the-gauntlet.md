# Escher — The Gauntlet
> Reconstructed from Codex task [019f9c71-cb1d-77f1-ba62-522af34abcd8](thread://019f9c71-cb1d-77f1-ba62-522af34abcd8). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge statement and supplied material

**Name:** Escher — The Gauntlet  
**Points:** 300  
**Author:** ravenspar

> The gate answers only in its own tongue, and it is not one machine. It shows you how it runs a handful of programs, but it never explains itself. Knock, and it names a nonce; reply as the machine would, four times running. The flag is not in the files.

- Service: `nc 104.211.98.123 1339`
- Handout: [Google Drive](https://drive.google.com/file/d/1c2BHiG09yuXyiEwo6aJzNsWa0C_E6uA8/view?usp=sharing)
- User-provided team ticket: `[REDACTED_TEAM_TICKET]`
- Local handout size: 6,478 bytes
- SHA-256: `B7972B0321280CEBA1E5B85FD26576B6E955678FA8B5CB1B3B5E584EC1625521`

## Category and status

- **Category:** Custom VM reverse engineering and interactive protocol solving.
- **Status:** **Confirmed solved.**
- **Confirmed team-specific flag:** `pbctf{7864cffed9bd95c7}`

## High-level solution overview

The handout supplies five tiny ESVM programs and full execution traces, but no instruction-set manual. Reconstruct each opcode from state transitions, implement an emulator, validate all samples, then select the live gate’s real two-module pipeline. The recovered manifest claims `whirl → sift`, but explicitly warns its front-end may be stale. The gate’s taunt confirms that `whirl` is wrong. The correct chain is `diffuse → sift`. With that pipeline and the user’s ticket, answer four fresh 16-byte nonces in one connection; the gate prints the team-derived flag.

## Detailed procedure

1. Parse the ESVM container:

   ```text
   bytes 0..4  = "ESVM" + version 01
   bytes 5..6  = code length in 3-byte instructions, little-endian
   bytes 7..8  = data length, little-endian
   then        = code
   then        = data section
   ```

2. Infer opcode semantics by comparing every trace step’s `before`, instruction bytes, `after`, input, output, and next program counter.

   Recovered instruction set:

   | Opcode | Semantics |
   |---:|---|
   | `0x01` | Load immediate: `R[a] = b` |
   | `0x02` | Table read into `R[a]` through pointer `R[b]`, then increment pointer |
   | `0x03` | Add modulo 256: `R[a] += R[b]` |
   | `0x04` | Reflected subtraction: `R[a] = R[b] - R[a]` |
   | `0x05` | XOR: `R[a] ^= R[b]` |
   | `0x06` | Append `R[a]` to output |
   | `0x07` | Branch to instruction `b` while `R[a] != 0` |
   | `0x08` | Set flag to unsigned `R[a] < R[b]` |
   | `0x09` | Branch to instruction `a` when comparison flag is false |
   | `0x0A` | Increment `R0` modulo 256 |
   | `0x0B` | Halt |
   | `0x0C` | Read one input byte into `R0` |

3. Implement eight 8-bit registers, one comparison flag, instruction-indexed PC, input/output streams, data bounds checks, and a step limit.

4. Validate all supplied samples exactly:

   ```text
   sample1 output                = 0300020100
   sample2(sample1 output)       = 0d0a0c0b0a
   sample3 output                = 0a02f4
   sample4 output                = 11223366
   sample5 output                = aa00
   ```

5. Implement module chaining:

   ```text
   output = stage2(stage1(nonce))
   ```

6. Initially test the manifest’s `whirl → sift`. The live gate responds:

   ```text
   that front-end is not the one behind this gate
   ```

   This is an intentional near-miss hint. The manifest itself says a front-end was swapped and may not have been updated.

7. Switch to `diffuse → sift`. A deterministic local probe is:

   ```text
   input  = 000102030405060708090a0b0c0d0e0f
   output = f0a561f733fec2fe4bda9ecbf0a56599
   ```

8. Implement the gate protocol:

   - Connect to `104.211.98.123:1339`.
   - Send ticket at `ticket:`.
   - Parse `nonce: <32 hex>`.
   - Run the two-stage pipeline.
   - If the gate announces `dialect: xx`, XOR every output byte with that byte.
   - Send the response as lowercase hex.
   - Repeat for four consecutive rounds.

9. Successful live transcript:

   ```text
   nonce ce9636db8416cebb27761f8ca3d4214f
   resp  be1f9dcf9bd341ca619585b4f078d09c

   nonce 647f2e8611045dc88889d50f946cc48d
   resp  5487937a3b00d0d2c25a47cfe107f989

   nonce 47ce3d28a1978acaf6066afacd06a26f
   resp  37b9b7eeb8bdfde84ad7d0c5ea9ecd8d

   nonce eae163f82bbef31136e4aa93a4ac6b87
   resp  da6fc2ec64f2defdc3cbf153f15a9686

   the gate swings open.
   pbctf{7864cffed9bd95c7}
   ```

## Important command

```powershell
python 'C:\Users\kingg\Desktop\codex\escher-gauntlet\solve_escher.py' `
  --handout 'C:\Users\kingg\Desktop\codex\escher-gauntlet\handout' `
  --stage1 diffuse `
  --ticket '[REDACTED_TEAM_TICKET]'
```

The current solver defaults to `whirl`; supplying `--stage1 diffuse` is essential.

## Evidence and validation

- All five supplied traces reproduced byte-for-byte before touching the oracle.
- The live taunt rejected only the front-end assumption, prompting the documented module pivot.
- The same live connection accepted four computed responses consecutively.
- The service itself printed `the gate swings open` and the flag.
- The README states that the flag is ticket-derived; this is the exact flag for the supplied team ticket.

## Failed approaches and decoys

- `whirl → sift` is the principal decoy. It comes from the stale manifest and produces the specific wrong-front-end taunt.
- The gate can shift to a dialect such as `0x5a` after repeated wrong probing. That XOR is a transport adaptation, not part of the underlying VM pipeline.
- A single correct round is insufficient; all four must pass in one session.
- No flag-like content in the handout can be authoritative because the challenge explicitly generates the flag behind the gate.

## Codex skills used

- `solve-challenge`
- `ctf-reverse`
- `ctf-misc`

## Tools, utilities, libraries, and services

- Python 3, `socket`, `argparse`, `dataclasses`, and regular expressions.
- PowerShell/ZIP extraction and file inspection.
- Raw TCP service at `104.211.98.123:1339`.
- Custom ESVM emulator and pipeline solver.

## Local artifacts

- `C:\Users\kingg\Desktop\codex\escher-gauntlet\handout.zip`
- `C:\Users\kingg\Desktop\codex\escher-gauntlet\handout`
- `C:\Users\kingg\Desktop\codex\escher-gauntlet\solve_escher.py`

## Diagram assessment

**Yes.** Suggested nodes and edges:

```text
sample programs + traces → opcode table → validated ESVM emulator
nonce → diffuse.bin → intermediate bytes → sift.bin → response
optional dialect byte → XOR response → gate
four accepted rounds → ticket-derived flag
manifest whirl path → live taunt → discarded
```

---

## Final High-Level Overview

The handout traces were used to reconstruct and verify the custom VM. The live service exposed the manifest front-end as a decoy, so the solver switched to diffuse then sift and answered all four ticketed nonce rounds in one connection.
