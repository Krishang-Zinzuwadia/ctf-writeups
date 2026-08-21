# PBCTF 5.0 — CTF Write-ups

This archive was reconstructed from three Codex tasks and their surviving local artifacts:

- `019f9d77-cc56-7923-8161-500eb8d68d16` — Telegram bot, Arkham Lockdown, The Moving Statue, Spiegel
- `019f9d2f-7242-7712-8c44-875a550c8e0c` — Yum Sea Pee, PB Lore Express
- `019f9cc8-4de0-73f3-b2fe-0be6b50a2937` — Sh*t Secrets Got Exposed, Northbank Health, CrushDatFTP
- `019f9cbe-3232-7381-84d0-42ac6e76a17e` — Snowfalls Snowfairy, Backfill, The Constellation, The Guard, Residual Evidence
- `019f9c71-cb1d-77f1-ba62-522af34abcd8` — Mimic, Cold Reading, VaultKey, Escher, Gatekeeper, Operation Chimera
- `019f9d81-75da-7f93-a454-14abd9c8c7a1` — Night Shift, Wonderland, Leviathan

The supplied screenshots were used only as supporting evidence for names, values, categories, and solved status where the mapping was unambiguous. They are preserved under [`evidence/`](evidence/).

## Integrity rule

A flag is marked **confirmed** only when it was returned by the challenge service, tied to the intended artifact and corroborated by the solved scoreboard, or otherwise independently validated. Rejected candidates and decoys are kept in warning sections and are never promoted to flags.

## Index

| Challenge | Category | Status | Confirmed flag |
|---|---|---|---|
| [PBCTF Telegram Registration Bot](01-telegram-registration-bot.md) | AppSec/Web, inferred | **ABORTED** | None |
| [Arkham Lockdown](02-arkham-lockdown.md) | pwn | **CONFIRMED SOLVED** | `pbctf{BATMAN_IS_NOT_GAY}` |
| [The Moving Statue](03-the-moving-statue.md) | Forensics/Web/Crypto, inferred | **PARTIALLY SOLVED / BROKEN** | None |
| [Spiegel — The Grimoire of Equivalent Spells](04-spiegel-grimoire-of-equivalent-spells.md) | Cryptography | **CONFIRMED SOLVED** | `pbctf{die_etiketten_waren_nie_zauber_me-jain-anurag}` |
| [Yum Sea Pee](05-yum-sea-pee.md) | AI Security/MCP, inferred | **UNSOLVED** | None |
| [PB Lore Express](06-pb-lore-express.md) | AI Security | **UNSOLVED** | None |
| [Sh*t Secrets Got Exposed](07-shit-secrets-got-exposed.md) | Forensics | **CONFIRMED SOLVED** | `pbctf{3f9d2c7a1e6b48d05c93a7f21e4d8b60}` |
| [Northbank Health](08-northbank-health.md) | Web Exploitation, inferred | **CONFIRMED SOLVED** | `pbctf{c0ncurr3nt_4pp34ls_0v3rr4n_th3_thr33_sl0t_l1m1t}` |
| [CrushDatFTP](09-crushdatftp.md) | Cloud/Web, inferred | **UNSOLVED / INCOMPLETE** | None |
| [Snowfalls Snowfairy / Way to Arctic](20-snowfalls-snowfairy.md) | Miscellaneous / Web / Forensics | **CONFIRMED SOLVED** | `ctf{wh1t3_f4c3_r3m3mb3r3d_th3_cub3_sp34ks}` |
| [Backfill / Interrupted Registry Migration](21-backfill.md) | OCI Forensics / Cryptography | **CONFIRMED SOLVED** | `pbctf{th3_c0s3ts_w3r3_th3_c4rg0_and_th3_h0l3s_w3r3_th3_k3y}` |
| [The Constellation](22-the-constellation.md) | Web Exploitation | **CONFIRMED SOLVED** | `pbctf{635c5d2b25a52322}` |
| [The Guard / Telegram Registration Bot](23-the-guard.md) | Web Exploitation | **UNRESOLVED** | None |
| [Residual Evidence](24-residual-evidence.md) | Forensics | **UNRESOLVED** | None |
| [Mimic](25-mimic.md) | Cryptography / Forensics | **UNRESOLVED / CHECKER MISMATCH** | None |
| [Cold Reading](26-cold-reading.md) | AppSec / Android / Web | **CONFIRMED SOLVED** | `pbctf{1d0r_by_pr3d1ct4bl3_1ds_1g}` |
| [VaultKey](27-vaultkey.md) | AppSec / Android / Cryptography | **CONFIRMED SOLVED** | `pbctf{C0NGR4TUL4T10NS_y0u_F0uND_1T_1N_7H3_N4T1v3_0R4cL3_5uP3r}` |
| [Escher — The Gauntlet](28-escher-the-gauntlet.md) | Reverse Engineering | **CONFIRMED SOLVED** | `pbctf{7864cffed9bd95c7}` |
| [Gatekeeper](29-gatekeeper.md) | pwn / Linux Forensics | **CONFIRMED SOLVED** | `pbctf{n0_sh3ll_1s_st1ll_4_sh3ll}` |
| [Operation Chimera :: Signal Fragment Two](30-operation-chimera-signal-fragment-two.md) | Misc / Cryptography / Web3 | **BROKEN AS DEPLOYED** | Claim word `REMANV`; no flag |
| [Night Shift](31-night-shift.md) | pwn / Linux Forensics | **CONFIRMED SOLVED** | `pbctf{v_f0r_v1ct0ry_th3n_sud0}` |
| [Wonderland — The Mad Tea-Party](32-wonderland-the-mad-tea-party.md) | Web Exploitation | **CONFIRMED SOLVED** | `pbctf{b5d492a6cd107d71}` |
| [Leviathan](33-leviathan.md) | Network Forensics / Cryptography | **UNRESOLVED** | None |

## Confirmed flags

```text
pbctf{BATMAN_IS_NOT_GAY}
pbctf{die_etiketten_waren_nie_zauber_me-jain-anurag}
pbctf{3f9d2c7a1e6b48d05c93a7f21e4d8b60}
pbctf{c0ncurr3nt_4pp34ls_0v3rr4n_th3_thr33_sl0t_l1m1t}
ctf{wh1t3_f4c3_r3m3mb3r3d_th3_cub3_sp34ks}
pbctf{th3_c0s3ts_w3r3_th3_c4rg0_and_th3_h0l3s_w3r3_th3_k3y}
pbctf{635c5d2b25a52322}
pbctf{1d0r_by_pr3d1ct4bl3_1ds_1g}
pbctf{C0NGR4TUL4T10NS_y0u_F0uND_1T_1N_7H3_N4T1v3_0R4cL3_5uP3r}
pbctf{7864cffed9bd95c7}
pbctf{n0_sh3ll_1s_st1ll_4_sh3ll}
pbctf{v_f0r_v1ct0ry_th3n_sud0}
pbctf{b5d492a6cd107d71}
```

## Diagrams

Selected multi-stage solutions materially benefit from diagrams:

- [`diagrams/spiegel-pipeline.excalidraw`](diagrams/spiegel-pipeline.excalidraw)
- [`diagrams/northbank-race.excalidraw`](diagrams/northbank-race.excalidraw)
- [`diagrams/shit-secrets-trail.excalidraw`](diagrams/shit-secrets-trail.excalidraw)
- [`diagrams/snowfalls-snowfairy-pipeline.excalidraw`](diagrams/snowfalls-snowfairy-pipeline.excalidraw)
- [`diagrams/backfill-recovery.excalidraw`](diagrams/backfill-recovery.excalidraw)
- [`diagrams/the-constellation-flow.excalidraw`](diagrams/the-constellation-flow.excalidraw)
- [`diagrams/operation-chimera-pipeline.excalidraw`](diagrams/operation-chimera-pipeline.excalidraw)
- [`diagrams/night-shift-escalation.excalidraw`](diagrams/night-shift-escalation.excalidraw)

The remaining write-ups are clearer as prose, tables, and compact payload examples.
