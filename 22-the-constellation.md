# The Constellation
> Reconstructed from Codex task [019f9cbe-3232-7381-84d0-42ac6e76a17e](thread://019f9cbe-3232-7381-84d0-42ac6e76a17e). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge record

- **Exact name:** `The Constellation`
- **Challenge ID/category/value:** ID 44, Web Exploitation, 400 points
- **Link:** <https://your-unique-name-evergreen-hollow-6858.fly.dev/>
- **Status:** **Confirmed solved**
- **Confirmed flag:** `pbctf{635c5d2b25a52322}`

The user supplied:

> **The Constellation**
> **400**
>  0  0
> A single star is not a constellation. Bring more of them into the dark and watch them reach for one another. This sky hides its chart. It was never painted on the page, so look instead at how the page reaches you. Learn the shape, then trace it back into the dark yourself. What the sky returns is sealed. But the chart you read was the key, so nothing stays hidden once you have counted the stars.
>
> [https://your-unique-name-evergreen-hollow-6858.fly.dev/](https://your-unique-name-evergreen-hollow-6858.fly.dev/)

## High-level solution

The star chart was delivered in an HTTP response header rather than HTML. Its base64 value decoded into six coordinate pairs. The application used a SharedWorker-backed session and WebSockets, with one star per connection. Six simultaneous sockets joined the same session and submitted a two-times-scaled version of the chart. The geometry unlocked a ticket-gated encrypted payload. A valid CTFd launch ticket obtained from `/hmac/ticket/44` opened the gate. The returned ciphertext decrypted with AES-256-CBC using `SHA256(chart ASCII)` as the key and a zero IV.

## Detailed procedure

1. **Read how the page arrived, not only its body.**

   The key response header was:

   ```text
   X-Star-Catalog: MTAsOTA7NzAsMjA7MTUwLDYwOzIxMCwxMDsxODAsMTIwOzkwLDE1MA==
   ```

   Base64 decoding produced:

   ```text
   10,90;70,20;150,60;210,10;180,120;90,150
   ```

   These were the six points of the hidden chart.

2. **Reverse the browser protocol.**

   The client used a SharedWorker to maintain a session. Each WebSocket first sent:

   ```json
   {"type":"hello","session":"<shared-session>","ticket":"<CTFd-ticket>"}
   ```

   A socket then placed its star using:

   ```json
   {"type":"pose","x":0,"y":0}
   ```

   One connection represented one star, so a constellation required six concurrent connections sharing the same session.

3. **Determine the coordinate transform.**

   Literal chart coordinates were rejected. The solver tested scales:

   ```text
   1, 1.5, 2, 2.5, 3, 4, 5
   ```

   and all four combinations of horizontal/vertical reflection around the viewport center `(600,450)`. It transmitted eight poses per trial to tolerate ordering or update timing. A scale factor of 2 produced the accepted shape.

4. **Pass the CTFd ticket gate.**

   Correct geometry without an authenticated ticket reached a final response asking the solver to start the challenge from CTFd. A generic credential endpoint:

   ```text
   /plugins/teamcreds/mine?challenge_id=44
   ```

   returned no usable credential. The correct route was:

   ```text
   /hmac/ticket/44
   ```

   The historical run received the short-lived ticket:

   ```text
   [REDACTED_TEAM_TICKET]
   ```

   This value is retained only as evidence; it is expired and should not be expected to work now.

5. **Run the six-star solver.**

   The historical invocation was:

   ```powershell
   $env:CONSTELLATION_TICKET='[REDACTED_TEAM_TICKET]'
   python .\solve_constellation.py
   ```

   The live service returned a 32-byte base64 ciphertext:

   ```text
   xsKkjTYduSles6G7KHrglK1LJKb6BXbj0HjdbPnfNIc=
   ```

6. **Derive the chart key and decrypt.**

   The AES-256 key was:

   ```text
   SHA256(b"10,90;70,20;150,60;210,10;180,120;90,150")
   ```

   Its hexadecimal value was:

   ```text
   e3786f3f06e7e266e527a80fae1422240f0f7b0a43561159b17133db0937f61c
   ```

   AES-256-CBC with a 16-byte all-zero IV and PKCS#7 unpadding returned:

   ```text
   pbctf{635c5d2b25a52322}
   ```

## Important endpoints, algorithms, and values

- Header: `X-Star-Catalog`
- Shared session protocol: `hello` followed by `pose`
- WebSocket target: the challenge origin over `wss://`
- Required concurrent stars: 6
- Successful coordinate scale: 2×
- Center used for tested reflections: `(600,450)`
- CTFd ticket route: `/hmac/ticket/44`
- Historical ticket: `[REDACTED_TEAM_TICKET]` (expired)
- Ciphertext: `xsKkjTYduSles6G7KHrglK1LJKb6BXbj0HjdbPnfNIc=`
- Cipher: AES-256-CBC
- Key: SHA-256 of the decoded chart text
- IV: 16 zero bytes

## Tools, libraries, services, and Codex skills

- **Codex skills used:** `solve-challenge`, `ctf-web`, `browser:control-in-app-browser`, `computer-use:computer-use`.
- **Tools/services:** in-app browser, the user's default Vivaldi browser, CTFd, Fly.io challenge service, PowerShell, Python.
- **Libraries/utilities:** `asyncio`, `websockets`, `hashlib`, PyCryptodome/`cryptography`, `curl`, source and header inspection, web and GitHub search.

## Validation evidence

- Six live WebSockets sharing one session caused the service to accept the chart geometry and return its sealed payload.
- The ticket came from the challenge's authenticated CTFd route.
- The specified chart-derived key produced valid PKCS#7 padding and the exact `pbctf{...}` plaintext under AES-CBC.
- Platform state later showed `solved_by_me: true`, although the original automation did not separately submit the flag during its solving pass.

## Rejected attempts and why

- **Raw chart coordinates:** did not match the server's expected viewport geometry.
- **One socket / one star:** the clue and protocol required six simultaneous connections.
- **Correct geometry without a ticket:** reached the intended final gate but was explicitly refused.
- **Generic team-credential route:** returned `null`; the HMAC ticket route was the relevant endpoint.
- **Alternative decryption families:** XOR, different hash inputs, other AES modes, and count-concatenated keys were tried. Only SHA-256 of the exact decoded chart plus a zero-IV AES-CBC decrypt produced valid padding and a flag.
- A user-triggered Escape stopped one Computer Use attempt; the investigation later resumed without changing the result.

## Local artifacts

| Artifact | Path |
|---|---|
| Final WebSocket solver | `C:\Users\kingg\Documents\codex 2\solve_constellation.py` |
| Saved page | `C:\Users\kingg\Documents\codex 2\index.html` |
| Client JavaScript | `C:\Users\kingg\Documents\codex 2\app.js` |
| Oracle/worker JavaScript | `C:\Users\kingg\Documents\codex 2\oracle.js` |
| Saved headers | `C:\Users\kingg\Documents\codex 2\headers.txt` |

## Diagram assessment

**A diagram would materially help.** Concise graph specification:

- Nodes: `HTTP response`, `X-Star-Catalog`, `decoded six points`, `six WebSockets`, `shared session`, `2x pose transform`, `ticket gate`, `sealed ciphertext`, `SHA256(chart)`, `AES-CBC`, `flag`.
- Edges: `response -> header -> points -> pose transform`; `six WebSockets -> shared session -> pose transform -> ticket gate`; `ticket gate -> ciphertext`; `points -> SHA256(chart) -> AES-CBC`, and `ciphertext -> AES-CBC -> flag`.

---

## Excalidraw diagram

Editable diagram: [The Constellation flow](diagrams/the-constellation-flow.excalidraw)

## Final High-Level Overview

A response header supplied six hidden points. Six ticketed WebSockets recreated their 2x-scaled geometry in one session, after which SHA-256 of the chart text decrypted the returned AES-CBC vault to the confirmed flag.
