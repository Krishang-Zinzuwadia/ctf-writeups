# Snowfalls Snowfairy / Way to Arctic
> Reconstructed from Codex task [019f9cbe-3232-7381-84d0-42ac6e76a17e](thread://019f9cbe-3232-7381-84d0-42ac6e76a17e). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge record

- **Exact platform name:** `Snowfalls Snowfairy`
- **Link:** <https://snowfall-mnemo.onrender.com/>
- **Platform category/value:** Miscellaneous, 250 points
- **Status:** **Confirmed solved**
- **Confirmed flag:** `ctf{wh1t3_f4c3_r3m3mb3r3d_th3_cub3_sp34ks}`

The user supplied this description:

> A snow fairy once lived inside MNEMO, and every face lay pure and white as fresh **snow**. Then came the fall — the snow scattered across everywhere, and the fairy's memory scattered with it.
>
> Gather the snow back. Coax it onto a single face until one side is pure snow again, and a hidden picture will surface from the static --- a fragment of all the fairy lost. [Way to Arctic](https://snowfall-mnemo.onrender.com/)

## High-level solution

The page was a remote Rubik's Cube oracle. Four rendered views were enough to reconstruct all 54 facelets and resolve the last ambiguous cubie placement through the cube's permutation parity. The resulting state was solved with `cubejs`'s Kociemba two-phase solver, and the moves were submitted to the live API. The service answered with `progress: 9`, `solved: true`, and a QR image. That QR led to a 3840×3840 PNG whose RGB-interleaved least-significant bits contained the flag beginning at byte offset 2.

## Detailed procedure

1. **Inspect the application and session protocol.**

   Browser inspection identified these endpoints:

   - `GET /api/new`
   - `POST /api/move`
   - `POST /api/rotate`
   - `POST /api/scramble`
   - `POST /api/reset`

   The page exposed ordinary face moves and visually hidden middle-slice moves corresponding to `M`, `E`, and `S`. A new session supplied a token that had to accompany subsequent state-changing requests.

2. **Collect enough views to identify the physical cube.**

   Four orientations of the rendered cube were captured. The colors were normalized, checked to occur exactly nine times each, and mapped into cubies:

   - all eight corner color triples were unique;
   - all twelve edge color pairs were unique;
   - the remaining ambiguity was a pair of down-layer edges;
   - only one placement satisfied a legal cube permutation parity.

3. **Serialize the reconstructed cube.**

   Using `U R F D L B` face order, the recovered facelet state was:

   ```text
   RULBURBFLFFBURLBRFDLDUFLUUULRRDDBDFUFDRDLBRRBDFUBBLLDF
   ```

4. **Solve the cube locally.**

   `cubejs@1.3.2`, which uses a Kociemba-style two-phase search, returned this 22-move solution:

   ```text
   U R' B D R D L' F2 U2 F U2 R2 B2 D L2 D F2 D R2 U2 L2 F2
   ```

   A Python `kociemba` route was also explored, but installing or initializing that path was slower and unnecessary once the direct `cubejs` package worked.

5. **Submit the moves to the live service.**

   Each move was sent sequentially to `/api/move` using the session token. The final response was at sequence 23 and included:

   ```json
   {
     "progress": 9,
     "solved": true,
     "message": "I remember now."
   }
   ```

   This is the first independent confirmation that the reconstructed state and move sequence were correct.

6. **Decode the service's solved image.**

   The result view included a QR code. `jsQR@1.4.0` decoded it to:

   <https://drive.google.com/file/d/1BLvkuGnxMojGXkYCKWbijQZalQQr5D1H/view?usp=sharing>

   The downloaded file was `mnemo_memory.png`, a 3840×3840 RGB poster referring to Atal Bihari Vajpayee's “गीत नया गाता हूँ.”

7. **Inspect the PNG beyond visible content.**

   Straightforward places for hidden data were negative:

   - no useful PNG text metadata;
   - no suspicious ancillary chunk payload;
   - no trailing payload after `IEND`;
   - no obvious plaintext in single-channel byte scans.

   The decisive extraction traversed pixels in RGB order, selected bit 0 of every channel, and packed bits most-significant-bit first. Its prefix was:

   ```text
   70626374667b77683174335f663463335f72336d336d6233
   ```

   Interpreted as bytes, the stream began `pbctf{...`; a `ctf{` match started at byte offset 2:

   ```text
   rgb-interleaved-b0-big hits:
   offset 2 -> ctf{wh1t3_f4c3_r3m3mb3r3d_th3_cub3_sp34ks}
   ```

## Important algorithms, endpoints, and commands

- Legal Rubik's Cube reconstruction using unique edge/corner color sets and parity.
- Kociemba two-phase solving through `cubejs@1.3.2`.
- Session endpoints: `/api/new`, `/api/move`, `/api/rotate`, `/api/scramble`, `/api/reset`.
- PNG extraction: RGB-interleaved channel order, least-significant bit, MSB-first byte packing.
- The exact solution submitted:

  ```text
  U R' B D R D L' F2 U2 F U2 R2 B2 D L2 D F2 D R2 U2 L2 F2
  ```

## Tools, libraries, services, and Codex skills

- **Codex skills used:** `solve-challenge`, `ctf-web`, `ctf-forensics`, `browser:control-in-app-browser`.
- **Tools/services:** in-app browser and Playwright-style browser control, Node.js, PowerShell, Python 3.14, Google Drive.
- **Libraries/utilities:** `cubejs@1.3.2`, `jsQR@1.4.0`, `sharp`, Pillow, NumPy.
- **Explored but not used for the final extraction:** Python `kociemba`, unavailable local `jsqr`, `qrcode-reader`, and `@zxing/library` packages.

## Validation evidence

The flag is confirmed by two linked results:

1. The remote cube service accepted the reconstructed cube solution and returned `progress: 9`, `solved: true`, and `"I remember now."`
2. The QR-delivered PNG's RGB-LSB stream contained the exact flag at a reproducible byte offset.

## Rejected leads and why

- A partially normalized QR view looked as if it might provide a shortcut, but the reliable path required a legal physical cube reconstruction and live solve.
- PNG metadata, ancillary chunks, bytes after `IEND`, and naive color-channel scans contained no flag.
- The two leading bytes `pb` in the LSB stream were not promoted as part of the flag; the actual flag delimiter `ctf{...}` began at offset 2 and closed normally.

## Local artifacts

| Artifact | Path / digest |
|---|---|
| Solved cube capture | `C:\Users\kingg\AppData\Local\Temp\mnemo-solved-cube.png` |
| Cube capture SHA-256 | `c72036890a9bb710b54caa6da0ae28d4a869360dca11e55bcc7c4213b0f69c77` |
| Solved QR capture | `C:\Users\kingg\AppData\Local\Temp\mnemo-solved-qr.png` |
| QR capture SHA-256 | `a6531b74ba8d1ade189181ab584e8dcead5aab24ea13140eef8b97094d381ab3` |
| Memory image | `C:\Users\kingg\AppData\Local\Temp\mnemo_memory.png` |
| Memory image SHA-256 | `a93695eca0fae2bfe1df735485d9fed5f279c4a51e27d767595e032f908044d4` |
| Downloaded QR decoder | `C:\Users\kingg\AppData\Local\Temp\mnemo-jsQR.js` |
| Direct cubejs workspace | `C:\Users\kingg\AppData\Local\Temp\mnemo-cubejs-direct\` |
| Abandoned Python solver workspace | `C:\Users\kingg\AppData\Local\Temp\mnemo-kociemba\` |

The memory PNG was 2,917,456 bytes; the cube and QR captures were 8,173 and 3,132 bytes respectively.

## Diagram assessment

**A diagram would materially help.** Concise graph specification:

- Nodes: `remote session`, `four cube views`, `cubie/parity reconstruction`, `54-facelet string`, `cubejs/Kociemba`, `/api/move sequence`, `solved QR`, `Drive PNG`, `RGB-LSB stream`, `flag`.
- Directed edges: `session -> views -> reconstruction -> facelet string -> solver -> move API -> QR -> PNG -> LSB stream -> flag`.
- Validation annotation: attach `progress:9, solved:true` to the `/api/move sequence -> solved QR` edge.

---

## Excalidraw diagram

Editable diagram: [Snowfalls Snowfairy / Way to Arctic flow](diagrams/snowfalls-snowfairy-pipeline.excalidraw)

## Final High-Level Overview

Four cube views were converted into a legal 54-facelet state, solved through the live move API, and validated by the service response. The solved QR led to a PNG whose RGB-interleaved least-significant-bit stream contained the flag.
