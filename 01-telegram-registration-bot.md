# PBCTF Telegram Registration Bot

## Challenge identity

- **Official name:** Not supplied in the task
- **Retrospective label:** PBCTF 5.0 Telegram registration bot / `@TheTelegrafBot`
- **Points:** Not supplied
- **Category:** AppSec/Web, inferred
- **Status:** **ABORTED**
- **Confirmed flag:** None

## Original question

> Shaurya made a Telegram bot for PBCTF 5.0 registrations, but it did come with major vulnurability.
>
> https://t.me/TheTelegrafBot
>
> solvr this ctf asap

No host, port, or downloadable attachment was supplied.

## High-level overview

The task attempted to inspect the bot’s public profile, command surface, and an existing Telegram conversation. The public page exposed only the description “Bot for Back-end Functions.” Telegram Web was not authenticated. An existing Vivaldi bot tab was detected, but desktop control was stopped before the chat could be opened or any command sent.

Because no interaction occurred, no vulnerability was demonstrated and no flag was obtained.

## Procedure performed

1. Search the public web for `TheTelegrafBot`, PBCTF references, and links to Shaurya.
2. Open `https://t.me/TheTelegrafBot`.
3. Open `https://web.telegram.org/k/#@TheTelegrafBot`.
4. Check for a public command list or linked registration backend.
5. Confirm Telegram Web was not signed in.
6. Enumerate available browser/desktop windows.
7. Locate an existing Vivaldi tab containing a bot-related conversation.
8. Attempt to activate and inspect that tab.
9. Stop when desktop automation was cancelled. No message was sent.

## Skills and workflows used

- `solve-challenge`
- Telegram/web-exploitation workflow
- `browser:control-in-app-browser`
- `computer-use:computer-use`

## Tools and services used

- Web search
- Telegram public pages
- Telegram Web
- In-app browser control
- Node REPL browser orchestration
- Windows desktop automation
- Vivaldi

## Evidence and validation

- The public profile description was visible.
- Telegram Web required an authenticated session.
- An existing bot-related Vivaldi tab was found.
- No bot command, payload, exploit result, or flag was produced.

## Files created

None attributable to this challenge.

## Rejected or unconfirmed candidates

None.

## Diagram

No diagram is needed because the investigation ended before a meaningful exploit chain was established.
