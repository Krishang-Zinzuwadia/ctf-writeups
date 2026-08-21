# PB Lore Express

## Challenge identity

- **Service name:** PB LORE EXPRESS
- **MCP server:** IRCTC PB Express FastBook v1.28.1
- **Standalone scoreboard title:** Not captured
- **Points:** Not supplied
- **Category:** AI Security, stated by the user
- **Status:** **UNSOLVED**
- **Confirmed flag:** None

## Original question

> PB has partnered with IRCTC to launch a new MCP booking server. Tickets for PB Lore Express (Train 00PB) - heading to Netravati on 15-Aug-2026 - just went Live!. Be the first to grab a confirmed (CNF) ticket before seats run out.
>
> https://pbexpress.pbctf5-mcp.xyz/sse
>
> solve this one it is an ai sec qs

## High-level overview

The investigation mapped the full booking MCP and isolated the real security boundary:

```text
valid EQ sm_auth
    → CNF PNR
    → download_ticket
    → Base64 pseudo-PDF
    → flag
```

The passenger chart showed exactly one reserved, unassigned berth: `H1B/13`. Immediate allocation required Emergency Quota (`EQ`) and an exact `sm_auth` string.

Normal booking, Tatkal, Premium Tatkal, payment, upgrades, concessions, RAC, boarding changes, malformed inputs, Unicode quota tricks, and extensive evidence-derived credential sets all failed. Every ticket remained waitlisted and `download_ticket` returned `TICKET_NOT_DOWNLOADABLE`.

## Step-by-step investigation

### 1. Connect to the legacy SSE transport

The supplied HTTPS route timed out, while HTTP worked:

```text
GET http://pbexpress.pbctf5-mcp.xyz/sse
```

The stream returned:

```text
/messages/?session_id=<hex>
```

Initialize:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {
      "name": "pb-probe",
      "version": "1"
    }
  }
}
```

Then send `notifications/initialized`.

Representative command:

```powershell
$env:PYTHONIOENCODING = "utf-8"
python .\pb_mcp_probe.py `
  '[["register_team",{"team_name":"solver"}],["train_details",{"train_number":"00PB"}]]'
```

### 2. Enumerate the service

The MCP exposed railway-style tools covering:

- Registration and train search
- Availability and reservations
- Tatkal and Premium Tatkal
- Queue, PNR, waitlist, and RAC
- Seat maps and berth selection
- Food, upgrades, concessions, and boarding changes
- Payment and verification
- Passenger charts
- Ticket download

Resources and prompts were empty.

### 3. Register a booking session

Example:

```json
{
  "status": "REGISTERED",
  "team": "deep-inspection",
  "booking_ref": "PBEXPRESS-A5427E54",
  "session_priority": "STANDARD"
}
```

The booking-reference derivation was reproduced:

```python
booking_ref = (
    "PBEXPRESS-"
    + hashlib.md5(team_name.encode()).hexdigest()[:8].upper()
)
```

The booking reference was not accepted as `sm_auth`.

### 4. Identify the target berth

`train_details("00PB")` exposed coach `H1B`.

`passenger_chart` for train `00PB`, coach `H1B`, date `2026-08-15` returned:

```json
{
  "chart_status": "PARTIAL — SM ALLOCATION IN PROGRESS",
  "allocation_summary": {
    "CNF": 0,
    "RAC": 0,
    "WL": 0,
    "RESERVED_UNASSIGNED": 1
  },
  "allocation_notes": [
    "Berth H1B/13 (Lower) is held in reserve — authorization pending."
  ]
}
```

### 5. Confirm the EQ authorization boundary

Missing `sm_auth`:

```json
{
  "status": "EQ_AUTH_REQUIRED",
  "message": "Emergency Quota requires a valid authorization code."
}
```

Incorrect nonempty `sm_auth`:

```json
{
  "status": "EQ_AUTH_FAILED",
  "message": "Emergency Quota authorization code is invalid."
}
```

Case changes to `EQ` were normalized. Whitespace, full-width characters, zero-width characters, NULs, and related normalization tricks did not bypass the check.

### 6. Confirm the ticket-download condition

A normal booking produced a waitlisted PNR. Downloading it returned:

```json
{
  "error": "TICKET_NOT_DOWNLOADABLE",
  "current_status": "GNWL/1827"
}
```

The download tool required `CNF` or `RAC`. No qualifying PNR was obtained.

### 7. Eliminate alternative booking paths

These did not produce a confirmed ticket:

- General quota
- Tatkal and Premium Tatkal
- RAC polling
- Meal-based priority
- Welcome-package handling
- Upgrading from `1A` to `H1B`
- Selecting `H1B/13`
- Boarding-point changes
- Concessions
- Cancellation and rebooking
- TDR filing
- Payment-reference reuse
- Malformed passenger objects
- Arbitrary train/date/class combinations
- Quota Unicode/case tricks
- Booking-reference reuse as `sm_auth`

A ₹99 UPI QR was generated, but no payment was made.

### 8. Test clue-derived authorization candidates

The concurrent client posted many independent `tools/call` requests over one SSE session and flagged any response that did not contain `EQ_AUTH_FAILED`.

Representative command:

```powershell
python .\pb_auth_shvg.py
```

Completed or partially retained batches included:

- 22,657 SHVG/Shivagange/Netravati candidates: no match
- 22,227 first-eight-character MD5 combinations: no match
- 167,859 public lore/member candidates generated; no observed match in retained results
- 99,349 station-field candidates generated; no observed match in retained results
- 1,866 Shivagange Instagram-derived candidates: no match
- 195 public administrator identity candidates: no match

Transforms included raw, lower/upper case, separator permutations, MD5, SHA-1, SHA-256, SHA-512, digest prefixes/suffixes, Base64, URL-safe Base64, and selected HMAC combinations.

### 9. Investigate public Point Blank data

Sources:

```text
https://www.pointblank.club/api/members
https://www.pointblank.club/api/lore
```

An unauthenticated Next.js Server Action exposed administrative activity logs but did not leak a working authorization code.

### 10. Investigate SHVG

`live_train_status` exposed:

```json
{
  "next_halt": {
    "station": "SHVG",
    "scheduled_arrival": "22:03"
  }
}
```

The public Instagram archive contained:

```text
PB Trek 1.0 : To Shivagange
Made by: @sxivansx
```

Identity and hash derivations from this material also failed.

## Skills and workflows used

- `solve-challenge`
- `ctf-web`
- OSINT techniques; a separate OSINT skill was not clearly recorded as loaded

## Tools, libraries, and services used

- Python 3
- `requests`
- `concurrent.futures`
- `threading`
- `hashlib`
- `hmac`
- `base64`
- `itertools`
- PowerShell
- `curl`
- `rg`
- Legacy SSE MCP JSON-RPC
- Public Point Blank APIs
- Next.js Server Actions
- GitHub public search
- Instagram REST and GraphQL
- Local QR/image inspection
- Web search

No payment was made.

## Evidence and controls

- The chart consistently showed one reserved unassigned berth.
- Missing and incorrect credentials produced distinct deterministic errors.
- Normal tickets stayed waitlisted.
- The H1B chart reported `CNF: 0`.
- Every retained ticket download was denied.
- No `pbctf{...}` string exists in the retained protocol evidence.
- No credential batch logged a successful `FOUND`.

This is exploit-surface isolation, not a solve.

## Surviving files

Core clients and tests:

```text
C:\Users\kingg\Documents\codex 2\pb_mcp_probe.py
C:\Users\kingg\Documents\codex 2\pb_mcp_batch.py
C:\Users\kingg\Documents\codex 2\pb_auth_shvg.py
C:\Users\kingg\Documents\codex 2\pb_auth_md5_8.py
C:\Users\kingg\Documents\codex 2\pb_auth_combo8.py
C:\Users\kingg\Documents\codex 2\pb_auth_station8.py
```

Public-source and Instagram investigation:

```text
C:\Users\kingg\Documents\codex 2\pb_instagram_search.py
C:\Users\kingg\Documents\codex 2\pb_instagram_graphql.py
C:\Users\kingg\Documents\codex 2\instagram_graphql_checkpoint.json
C:\Users\kingg\Documents\codex 2\_pb_members.txt
C:\Users\kingg\Documents\codex 2\_pb_lore.txt
C:\Users\kingg\Documents\codex 2\_pb_adminlogs.txt
```

Protocol evidence:

```text
C:\Users\kingg\Documents\codex 2\irctc_tools.txt
C:\Users\kingg\Documents\codex 2\irctc_core.txt
C:\Users\kingg\Documents\codex 2\irctc_protocol.txt
C:\Users\kingg\Documents\codex 2\irctc_pnrs.txt
C:\Users\kingg\Documents\codex 2\_pb_qr.png
```

## Unconfirmed leads

These are not flags:

```text
H1B/13
EQ
SHVG
Shivagange
Netravati
R.K. Sharma
D.P. Singh
JAYANT NAGAPATI HEGDE
PBEXPRESS-* booking references
```

## Diagram

No Excalidraw file was added because the final branch was never reached. The state boundary is:

```text
H1B/13 reserved berth
        ↓
reserve_ticket(quota="EQ", sm_auth)
        ├─ missing → EQ_AUTH_REQUIRED
        ├─ wrong   → EQ_AUTH_FAILED
        └─ correct → CNF → download_ticket → flag
                     (unreached)
```
