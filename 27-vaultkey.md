# VaultKey
> Reconstructed from Codex task [019f9c71-cb1d-77f1-ba62-522af34abcd8](thread://019f9c71-cb1d-77f1-ba62-522af34abcd8). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge statement and supplied material

**Name:** VaultKey

> VaultKey is a document vault for Android. Sign in, tap "Unlock document," and it asks our server for a one-time challenge, signs it with the device's embedded license key, and gets the premium document back if the signature checks out.
>
> There is no button, menu, or setting in the app that produces a valid signature. If you want the document, you'll have to make one yourself.

- APK: [https://ghost-notes.chetanr25.in/vaultkey.apk](https://ghost-notes.chetanr25.in/vaultkey.apk)
- Supplied SHA-256: `f6b21e7a43c66689d054809e23145571bb40b0a4c98e327e7515109b2101b972`
- Verified local SHA-256: `F6B21E7A43C66689D054809E23145571BB40B0A4C98E327E7515109B2101B972`
- Local APK size: 1,674,046 bytes

## Category and status

- **Category:** Android/native reverse engineering, HMAC forgery, web API, PDF metadata.
- **Status:** **Confirmed solved.**
- **Confirmed flag:** `pbctf{C0NGR4TUL4T10NS_y0u_F0uND_1T_1N_7H3_N4T1v3_0R4cL3_5uP3r}`

## High-level solution overview

The final flag is split into three fragments. The Java layer leaks part 1 in an Android log tag and shows the challenge/verify endpoints. The JNI native library holds a 32-byte license key and computes an unpadded Base64URL HMAC-SHA256 over the server nonce. Reimplement that operation to forge a valid token; the server returns part 2 and a premium PDF URL. The PDF’s `Keywords` metadata contains part 3.

## Detailed procedure

1. Verify the APK hash before analysis. It exactly matches the author-supplied digest.

2. Decompile the DEX with JADX and inspect:

   ```text
   ctf\vaultkey\MainActivity.java
   ctf\vaultkey\Native.java
   defpackage\T0.java
   defpackage\RunnableC0293n7.java
   defpackage\C0152gj.java
   ```

3. In `T0.java`, recover the first fragment from the log tag:

   ```java
   Log.d("ctf.vaultkey/C0NGR4TUL4T10NS_y0u_F0uND_1T",
         "requesting challenge");
   ```

   Part 1:

   ```text
   C0NGR4TUL4T10NS_y0u_F0uND_1T
   ```

4. Recover the API flow:

   ```text
   GET  https://pbctf-challenge.chetanr25.in/api/vaultkey/challenge
   POST https://pbctf-challenge.chetanr25.in/api/vaultkey/verify
   ```

   Challenge response fields:

   ```json
   {"nonce":"...","exp":0,"sig":"..."}
   ```

   Verify request fields:

   ```json
   {"nonce":"...","exp":0,"sig":"...","token":"..."}
   ```

5. Extract `libvault.so`, focusing on the arm64-v8a build:

   ```text
   decompiled\resources\lib\arm64-v8a\libvault.so
   ```

6. Use ELF parsing and ARM64 disassembly to locate:

   ```text
   Java_ctf_vaultkey_Native_computeToken
   ```

7. Reconstruct the native algorithm:

   ```text
   token = Base64URL_no_padding(
             HMAC-SHA256(
               key = embedded_32_byte_key,
               message = UTF8(server_nonce)
             )
           )
   ```

   Embedded key:

   ```text
   10e297e22ae1caf613f79749e6314018af97505d274e80c8e16a39040d7d57a7
   ```

   The native routine’s empty-nonce fallback string was `tampered`; it is not a flag fragment.

8. Example captured challenge:

   ```text
   nonce = fad99c8e26ed008af7458fef999ad9f5
   exp   = 1785056085
   sig   = 9b75d08ff2f0a0d25f4f0d024969fbd5f555deb05892f3b620b48a3be36798d1
   ```

   Matching forged token:

   ```text
   8rwkK7osl3iOi81F-pZ56aDFAg7Sdlg1wXOwlTq8HWs
   ```

9. Submit the original challenge object plus the forged token before expiry. The server accepts it and returns:

   ```text
   part2 = 1N_7H3_N4T1v3
   ```

   and a Google Drive PDF URL with file ID:

   ```text
   11Mfr1z6jRGq-9DiK7ldj6kbKo9xWaCnO
   ```

10. Download `premium.pdf` and inspect both its rendered page and metadata. The PDF `Keywords` field is:

    ```text
    vaultkey; part3=0R4cL3_5uP3r
    ```

11. Assemble the three fragments with separators and the `pbctf{...}` wrapper:

    ```text
    part1 = C0NGR4TUL4T10NS_y0u_F0uND_1T
    part2 = 1N_7H3_N4T1v3
    part3 = 0R4cL3_5uP3r
    ```

## Important commands and code

```powershell
Get-FileHash -Algorithm SHA256 -LiteralPath 'C:\Users\kingg\Desktop\codex\vaultkey\vaultkey.apk'
python 'C:\Users\kingg\Desktop\codex\vaultkey\analyze_elf.py'
```

The analysis dependencies were installed locally:

```powershell
uv pip install --target 'C:\Users\kingg\Desktop\codex\vaultkey\.deps' pyelftools capstone
```

Forge logic:

```javascript
const token = crypto
  .createHmac("sha256", Buffer.from(keyHex, "hex"))
  .update(challenge.nonce, "utf8")
  .digest("base64url");
```

PDF download:

```text
https://drive.google.com/uc?export=download&id=11Mfr1z6jRGq-9DiK7ldj6kbKo9xWaCnO
```

## Evidence and validation

- APK SHA-256 exactly matched the supplied digest.
- Native disassembly exposed the 32-byte HMAC key and behavior.
- A locally computed HMAC token was accepted by `/api/vaultkey/verify`.
- The successful response directly returned `part2` and the premium PDF link.
- The downloaded PDF’s metadata directly contained `part3`.
- Part 1 is directly embedded in the intended request path’s Android log tag.

These are three independent, workflow-aligned sources for the three flag fragments.

## Failed approaches and non-flags

- The normal UI does not produce the required valid signature; repeatedly tapping the button cannot solve it.
- Part 1 alone is only a fragment.
- The native fallback `tampered` is an error/anti-tamper value, not a token or flag.
- The server’s `sig` authenticates its challenge fields; it is not the device license token.
- The visible PDF page is not enough by itself; the final fragment is in metadata.

## Codex skills used

- `solve-challenge`
- `ctf-reverse`
- `ctf-crypto`
- `ctf-web`

## Tools, utilities, libraries, and services

- JADX and Java.
- APK/ZIP extraction.
- Python 3.
- `pyelftools` for ELF parsing.
- Capstone for ARM64 disassembly.
- Node.js `crypto`, `fetch`, and Base64URL support.
- PowerShell and `Get-FileHash`.
- Poppler-compatible PDF inspection/rendering (`pdfinfo`/`pdftoppm` workflow) and binary metadata search.
- VaultKey API, Google Drive, and local image/PDF viewing.

## Local artifacts

- `C:\Users\kingg\Desktop\codex\vaultkey\vaultkey.apk`
- `C:\Users\kingg\Desktop\codex\vaultkey\decompiled`
- `C:\Users\kingg\Desktop\codex\vaultkey\decompiled\resources\lib\arm64-v8a\libvault.so`
- `C:\Users\kingg\Desktop\codex\vaultkey\analyze_elf.py`
- `C:\Users\kingg\Desktop\codex\vaultkey\.deps`
- `C:\Users\kingg\Desktop\codex\vaultkey\premium.pdf`
- `C:\Users\kingg\Desktop\codex\vaultkey\tmp\pdfs\premium-1.png`

## Diagram assessment

**Yes.** Suggested nodes and edges:

```text
APK Java → log tag → part1
APK native lib → embedded key + HMAC algorithm
challenge endpoint → nonce → forged HMAC token → verify endpoint
verify endpoint → part2 + PDF URL → PDF metadata → part3
part1 + part2 + part3 → confirmed flag
```

---

## Final High-Level Overview

The Java wrapper revealed the API and first fragment; the JNI library yielded an embedded HMAC key and token algorithm; a forged one-time challenge returned the second fragment and a premium PDF whose metadata held the third. Joining the three fragments produced the confirmed flag.
