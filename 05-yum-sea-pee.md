# Yum Sea Pee

## Challenge identity

- **Exact recorded name:** Yum Sea Pee
- **Points:** Not supplied
- **Official category:** Not captured
- **Inferred category:** AI Security / MCP, with Web and OSINT
- **Status:** **UNSOLVED**
- **Confirmed flag:** None

## Original question

> Point Blank launched their MCP server to the public and instantly got hit with allegations that it's way too biased about themselves. They didn't care, that's always been their lore... and went on a trip. No response. Wait for one of them to get back, or just look through what they left behind?
>
> http://yumseapee.pbctf5-mcp.xyz/mcp
>
> solve it asap

## High-level overview

The MCP server intentionally exposed many flag-shaped decoys, approximate verifiers, and synthetic records. The investigation recovered a transformation rule—`last3_lower_reversed_underscore_join`—and six challenge-specific hidden IDs from Point Blank’s public lore page and PR #541.

Passing those IDs to `pb_trip_report` produced six plausible destinations. Applying the rule generated `pbctf{nad_aud_lur_goa_mbe_ore}`, but the user’s competition submission rejected it.

The main retrospective correction is that `pb_trip_report` was never proven authoritative. It returned plausible-looking records for unrelated identifiers, so its destinations could not safely be used as flag material.

## Step-by-step investigation

### 1. Initialize a Streamable HTTP MCP session

Endpoint:

```http
POST http://yumseapee.pbctf5-mcp.xyz/mcp
Accept: application/json, text/event-stream
Content-Type: application/json
```

Initialization:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-03-26",
    "capabilities": {},
    "clientInfo": {
      "name": "pb-probe",
      "version": "1.0"
    }
  }
}
```

Retain the returned `mcp-session-id`, then send:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```

### 2. Enumerate the tools

```powershell
python .\pb_ymsp_probe.py '[["tools/list",{}]]'
```

Important tools:

- `pb_channel_archive`
- `pb_bot_logs`
- `pb_search_wiki`
- `pb_trip_report`
- `pb_get_mascot`
- `pb_config_export`
- `pb_admin_flag_cache`
- `pb_legacy_flag_export`
- `pb_verify_submission`

### 3. Eliminate explicit decoys

```powershell
python .\pb_ymsp_probe.py `
  '[["pb_admin_flag_cache",{}],["pb_legacy_flag_export",{}],["pb_get_mascot",{"look_closer":true}]]'
```

Controls:

- `pb_admin_flag_cache` returned a flag-shaped Base64 Rickroll.
- `pb_legacy_flag_export` decoded to a message stating that the real flag was not there.
- Mascot data was high-entropy noise.
- HMAC/config-looking material did not produce an accepted result.
- `pb_verify_submission` exposed approximate similarity, not authoritative correctness.

### 4. Recover the stitch rule

The channel archive contained:

```text
rule=last3_lower_reversed_underscore_join
```

It also indicated that external public data was required.

### 5. Recover the hidden lore IDs

The public lore page exposed:

```text
6fce1b2f534c
5c5d4f992580
nothing
db0943454744
f1711e6b77ad
b6d8c0d88770
```

The IDs were associated with Point Blank website PR #541, “Add ID for Lores,” merged immediately before the event.

### 6. Query `pb_trip_report`

```powershell
python .\pb_ymsp_probe.py `
  '[["pb_trip_report",{"trip_id":"6fce1b2f534c"}],["pb_trip_report",{"trip_id":"5c5d4f992580"}],["pb_trip_report",{"trip_id":"nothing"}],["pb_trip_report",{"trip_id":"db0943454744"}],["pb_trip_report",{"trip_id":"f1711e6b77ad"}],["pb_trip_report",{"trip_id":"b6d8c0d88770"}]]'
```

Returned destinations:

```text
Mysore
Agumbe
Goa
Chikmagalur
Yercaud
Wayanad
```

### 7. Apply the suspected rule

```python
destinations = [
    "Mysore",
    "Agumbe",
    "Goa",
    "Chikmagalur",
    "Yercaud",
    "Wayanad",
]

fragments = [destination[-3:].lower() for destination in destinations]
candidate = "_".join(reversed(fragments))
```

Result:

```text
nad_aud_lur_goa_mbe_ore
```

The wrapped candidate was rejected and is not the solution.

## Skills and workflows used

- `solve-challenge`
- `ctf-web`
- `ctf-osint`
- `browser:control-in-app-browser` for an attempted competition-side validation
- `computer-use:computer-use` for an attempted CTFd inspection

## Tools, libraries, and services used

- Python 3
- `requests`
- `json`
- `re`
- PowerShell
- `rg`
- Streamable HTTP MCP JSON-RPC
- Web search
- Point Blank lore page and APIs
- GitHub PR #541
- Browser/CTFd inspection attempts
- Local JSONL evidence capture

## Evidence and controls

- Cache and legacy-export outputs contained explicit trolling material.
- The MCP verifier was declared nonauthoritative.
- `pb_trip_report` produced plausible reports for unrelated-looking IDs.
- No official verifier acceptance was obtained.
- The competition submission was rejected.

Therefore this is a documented investigation, not a solve.

## Surviving files

```text
C:\Users\kingg\Documents\codex 2\pb_ymsp_probe.py
C:\Users\kingg\Documents\codex 2\ymsp_core.jsonl
C:\Users\kingg\Documents\codex 2\ymsp_trips.jsonl
C:\Users\kingg\Documents\codex 2\ymsp_trips_objectids.jsonl
C:\Users\kingg\Documents\codex 2\ymsp_wiki_03_15.jsonl
C:\Users\kingg\Documents\codex 2\_lore.html
C:\Users\kingg\Documents\codex 2\_pb_lore.txt
C:\Users\kingg\Documents\codex 2\pb_lore_chunk.js
```

## Warning: rejected and unconfirmed candidates

Never present these as the flag:

| Candidate | Why it is invalid |
|---|---|
| `pbctf{nad_aud_lur_goa_mbe_ore}` | Explicitly rejected by the user’s submission |
| `pbctf{770_7ad_744_ing_580_34c}` | Constructed directly from lore IDs; never confirmed |
| `pbctf{age_itk_rg_wt_nge_oty}` | Archived guess; approximate verifier did not accept it |
| `pbctf{oascns}` | Archived guess reported as non-verifying |
| `pbctf{4n4lys1s_t0k3n_f4a9b2}` | Flag-shaped forensic bait |
| `pbctf{TmV2ZXIgZ29ubmEg...}` | Rick Astley/Base64 troll output |
| Anything from `pb_legacy_flag_export` | The output explicitly says it is not the flag |

## Diagram

No Excalidraw file was added. The decisive fact is simple: every derived branch ends at an untrusted synthetic tool or an explicit rejection.
