# The Guard / Telegram Registration Bot
> Reconstructed from Codex task [019f9cbe-3232-7381-84d0-42ac6e76a17e](thread://019f9cbe-3232-7381-84d0-42ac6e76a17e). Confirmed flags require intended-workflow evidence; rejected candidates remain explicitly marked.

## Challenge record

- **Exact platform name:** `The Guard`
- **Challenge ID/category/value:** ID 13, Web Exploitation, 300 points
- **Telegram bot:** <https://t.me/TheTelegrafBot>
- **Display name observed:** `kthnxbye`
- **Status:** **Unresolved / unfinished**
- **Confirmed flag:** **None**
- **Platform state during the run:** `solved_by_me: false`, zero submitted attempts

The user supplied:

> Shaurya made a Telegram bot for PBCTF 5.0 registrations, but it did come with major vulnurability.
>
> [https://t.me/TheTelegrafBot](https://t.me/TheTelegrafBot)

The user then confirmed Telegram was open in the default browser and authorized interaction with the bot.

## High-level investigation

The bot was started successfully and reported that it was parked. Its “Source Code!” button led to the PBCTF registration site. Reconnaissance found a public Firebase project and, more importantly, the exact public repository and historical source. An old login route leaked the string `pbstruggles`, suggesting a former admin registration secret. The current server, however, required a server-side `ADMIN_CODE` plus a valid reCAPTCHA v3 token. Firebase identity alone did not create the MongoDB user required by `/api/user/flag`, and Google reCAPTCHA could not be completed in the automated browser. No live privileged registration and no flag were obtained.

## Detailed procedure and evidence

1. **Interact with the actual Telegram bot.**

   The correct account was `@TheTelegrafBot`. After the explicit user authorization, START was sent. The bot's response indicated that it was parked and exposed a `Source Code!` button.

2. **Follow the source/registration link.**

   The button led to:

   <https://pbctf.pointblank.club>

   Saved frontend code revealed authenticated routes including:

   ```text
   /api/user/flag
   /api/me/bootstrap
   /api/registration
   /api/admin/register
   ```

3. **Test the public identity surface.**

   The site used a public Firebase project. A throwaway Firebase account could be created and an identity token obtained, but the application separately required a corresponding server-side MongoDB user record. Therefore a valid Firebase ID token alone did not pass `/api/user/flag`.

4. **Locate and inspect the public repository.**

   A live error exposed:

   <https://github.com/pointblank-club/pbctf>

   The default branch was `staging`. The repository was cloned into `guard_source`, then its current routes and git history were examined.

5. **Understand the current flag endpoint.**

   Current source showed that an authenticated user receives a user-specific flag:

   ```text
   pbctf{HMAC_SHA256(FLAG_SECRET, uid)[0:24]}
   ```

   Both `/api/user/flag` and `/api/me/bootstrap` relied on the backend user record and secret, so this was not locally forgeable from public values.

6. **Identify the intended registration barrier.**

   The normal `/api/registration` route called `isRegistrationClosed()` first and returned HTTP 403 while registrations were closed.

   The current `/api/admin/register` path instead required:

   - a server-side environment value `ADMIN_CODE`;
   - a valid reCAPTCHA v3 token for action `admin_register`;
   - valid user registration data.

7. **Audit git history for leaked secrets.**

   Commits including `80632df`, `3cff5cd`, and `3206f39` were inspected. Historical file recovery with:

   ```powershell
   git show 80632df:app/api/auth/login/route.ts
   ```

   revealed:

   ```javascript
   const SECRET_CODE = "pbstruggles";
   ```

   Older logic associated this code with an `@pointblank.club` administrative login flow. This made `pbstruggles` a credible exploit lead, but not a confirmed current `ADMIN_CODE`.

8. **Attempt the live admin registration route.**

   The public reCAPTCHA site key was:

   ```text
   6Lek-yctAAAAAO7pCksGE19JBus029D5_KIjnCjd
   ```

   A real page was opened under Playwright/Chromium to request an action token and submit:

   ```json
   {
     "name": "<throwaway>",
     "email": "<throwaway>",
     "password": "<throwaway>",
     "adminCode": "pbstruggles",
     "recaptcha_token": "<browser token>"
   }
   ```

   The Google reCAPTCHA host was unreachable in that environment, so no valid token was produced. A nonempty fake token caused the live endpoint to stall or time out rather than yielding a privileged account. The browser attempt was later interrupted with Escape.

## Important endpoints, secrets, and algorithms

- Bot: `@TheTelegrafBot`
- Registration site: `https://pbctf.pointblank.club`
- Public repository: `https://github.com/pointblank-club/pbctf`, branch `staging`
- User flag route: `/api/user/flag`
- Bootstrap route: `/api/me/bootstrap`
- Closed registration route: `/api/registration`
- Admin route: `/api/admin/register`
- Historical leaked code: `pbstruggles`
- reCAPTCHA site key: `6Lek-yctAAAAAO7pCksGE19JBus029D5_KIjnCjd`
- Per-user flag construction in current source:

  ```text
  pbctf{HMAC_SHA256(FLAG_SECRET, uid)[0:24]}
  ```

## Tools, libraries, services, and Codex skills

- **Codex skills used:** `solve-challenge`, `ctf-misc`, `ctf-web`, `computer-use:computer-use`, `playwright`.
- **Tools/services:** Telegram Web, Vivaldi, GitHub, `gh`, git, PowerShell, `curl`, Firebase Identity Toolkit REST, Playwright/Chromium, CTFd.
- **Techniques:** Next.js bundle/source review, route tracing, Firebase token testing, git history and deleted-source recovery, reCAPTCHA v3 action analysis.

## Why there is no confirmed flag

The run established source-level leads but never crossed the live authorization boundary:

- no MongoDB-backed application user was created through a permitted route;
- no valid reCAPTCHA token reached `/api/admin/register`;
- `pbstruggles` was never confirmed as the current server's `ADMIN_CODE`;
- `/api/user/flag` never returned a flag;
- no candidate flag was submitted or accepted.

Consequently, `pbstruggles` is evidence of a historical secret leak, not a flag and not proof of a current exploit.

## Rejected or incomplete approaches

- **Public Firebase signup:** created identity but not the required application database record.
- **Direct ordinary registration:** blocked because registration was closed.
- **Historical secret used as current admin code:** plausible but unverified due to the separate reCAPTCHA gate.
- **Fake/nonempty reCAPTCHA token:** timed out and did not prove acceptance.
- **General web/GitHub searches:** found no organizer write-up or confirmed flag.
- The three paid CTFd hints (IDs 37, 38, and 39; costs 20, 30, and 50) were not purchased, so they contributed no evidence.

## Local artifacts

| Artifact | Path |
|---|---|
| Saved Telegram page | `C:\Users\kingg\AppData\Local\Temp\telegrafbot.html` |
| Cloned repository | `C:\Users\kingg\Documents\codex 2\guard_source\` |
| Dashboard HTML | `C:\Users\kingg\Documents\codex 2\guard_dashboard.html` |
| Response headers | `C:\Users\kingg\Documents\codex 2\guard_headers.txt` |
| Saved JavaScript | `C:\Users\kingg\Documents\codex 2\guard_js\` |
| Login page | `C:\Users\kingg\Documents\codex 2\guard_login.html` |
| Registration page | `C:\Users\kingg\Documents\codex 2\guard_register.html` |

## Diagram assessment

**A diagram would materially help.** Concise graph specification:

- Nodes: `Telegram bot`, `source button`, `PBCTF site`, `public repository`, `historical pbstruggles leak`, `/api/admin/register`, `reCAPTCHA v3`, `MongoDB-backed user`, `/api/user/flag`, `per-user HMAC flag`.
- Edges: `bot -> source button -> site -> repository -> historical leak`; `historical leak -> /api/admin/register`; `reCAPTCHA v3 -> /api/admin/register`; `/api/admin/register -> MongoDB-backed user -> /api/user/flag -> per-user HMAC flag`.
- Mark the `reCAPTCHA v3 -> /api/admin/register` edge and everything downstream as **not completed**.

---

## Final High-Level Overview

The bot led to the registration application and its public repository. Git history leaked a plausible former admin code, but the live route still required reCAPTCHA and backend user provisioning. Because that boundary was never crossed, there is no confirmed flag.
