# Native Codex login through ego-browser

Use only when the effective CLI reports missing/unusable login, or the user
explicitly requests an account change. A logged-in ChatGPT browser session does
not establish CLI authentication. These steps follow the
[official Codex authentication documentation](https://learn.chatgpt.com/docs/auth).

Two login flows exist; pick the one that serves the failing request. Steps 1–6
below are native `codex login` (the physical main written to `~/.codex/auth.json`).
When `codexAccountMode` is `pool` (`ocx status --json`), the serving credential is
an OpenCodex Pool account, not the native login — use
[Pool account re-authentication](#pool-account-re-authentication-opencodex) at the
end of this file. That section also covers a revoked ChatGPT session, which
strands **every** Pool account that shares it, not just the main.

1. Check `codex login --help` and `codex login status` using the same executable,
   effective home, and credential-store configuration as the failing client.
   If already authenticated, diagnose routing or a demonstrated credential error;
   do not log out merely to exercise this procedure.
2. Read the installed **ego-browser** skill. Start `codex login` in a persistent
   terminal/PTY and retain its session handle. The native command owns the OAuth
   state, PKCE exchange, callback listener, and credential storage. Keep it alive.
3. Open the **exact fresh authorization URL emitted by that command** in
   ego-browser. Do not substitute the ChatGPT home page, replay an old URL, or
   construct a callback URL. Use the goal's existing TaskSpace if one exists.

For a new TaskSpace, supply that freshly emitted URL as the process environment
value `CODEX_LOGIN_URL` and run:

```sh
ego-browser nodejs <<'JS'
const oauthUrl = process.env.CODEX_LOGIN_URL;
if (!oauthUrl) throw new Error("Set CODEX_LOGIN_URL from the active codex login prompt");
const task = await taskSpace("Restore Codex login");
const page = task.page("p1");
await page.goto(oauthUrl);
console.log({ spaceId: task.spaceId, page: page.label });
console.log(await page.snapshot());
JS
```

4. Reuse the returned TaskSpace id and page label in subsequent invocations.
   Act on selectors from the latest snapshot using documented ego-browser APIs.
   Choose the intended existing account and workspace, completing the visible
   Codex authorization. If the task identifies a premium Main Account, verify
   that identity/workspace rather than accepting an arbitrary browser login.
5. For credentials the agent does not have, MFA, passkeys, or human verification,
   use ego-browser's handoff and let the user complete the sensitive step. Do not
   ask for passwords or one-time codes in chat. Resume that same flow after the
   user returns control. Never extract browser cookies/tokens into `auth.json`.
6. Observe the retained `codex login` process complete successfully and rerun
   `codex login status`. Browser success alone does not satisfy this check.
   Then run one minimal request through the user's actual CLI configuration.
   In OpenCodex, confirm the intended Main Account is usable and verify its
   serving-account label on the completed request.

## Pool account re-authentication (OpenCodex)

When `codexAccountMode` is `pool`, the serving credential is a Pool account.
Re-authenticate it in place with the OpenCodex flow, not (only) native
`codex login`, so the account keeps its id, priority, and log label:

```sh
ocx account login <provider> --id <account-id> --reauth --no-wait --json
```

`--no-wait --json` returns `{ flowId, url }` without blocking; the running proxy
holds the localhost callback listener. Save that `url` to a file and open the
**exact** value in ego-browser — never echo it, and never construct or replay a
URL; the `state`/PKCE URL is a short-lived credential. The browser's localhost
callback (default `localhost:1455`, or `--device` when nothing can reach it)
finishes the flow; a manual code goes back through `ocx account code <provider>
--flow <id>` read from stdin, never as an argument.

A single revoked ChatGPT session invalidates **every** Pool account that shares
it. The CLI shows `token_revoked` / `refresh_token_invalidated` — often surfaced
to the user as "Selected model is at capacity", which is not real capacity — and
`ocx account refresh <provider> --json` returns HTTP 401 or empty quota for the
affected accounts. Restoring one account does not restore the others. After the
main is usable, restore the rest:

1. Enumerate the pool with `ocx account list <provider> --json` and mark every
   account still unusable (401/empty quota, `needsReauth`, or a demonstrated
   request failure). `needsReauth: false` does not prove usability — a server-side
   revocation is not flagged locally; confirm against a refresh or a real request.
2. Re-authenticate each unusable account, **one flow at a time**: start its
   `--reauth` flow, complete it, then move to the next. Cancel a stale flow with
   `ocx account cancel <provider> --flow <id>` before starting another. Do not run
   an unbounded login/refresh loop.
3. Reuse the **same** ego-browser TaskSpace across all accounts. For each account,
   select its **exact** identity and workspace. Masked pickers hide the
   distinguishing part — two seats can both read `d***e@b***`; disambiguate by the
   full email and domain (for example a `data-identifier` attribute and its TLD)
   so a login is never bound to the wrong slot. Verify identity/workspace rather
   than accepting an arbitrary browser login.
4. Credentials the agent lacks, MFA, passkeys, or human verification → hand off in
   ego-browser and resume the same flow; never ask for passwords or codes in chat.
   An already-signed-in browser session may let account selection complete with no
   credential prompt; still confirm the exact identity before continuing.

Verify **each** restored account, not just the main: `ocx account refresh
<provider> --json` returns its quota without 401, and the active account is
confirmed by one minimal CLI request whose attempt-level serving label maps to
that account (`ocx account list` shows each seat's `logLabel`). Report any account
you could not restore as still unusable; never remove or overwrite a stored
account to hide a failed login.

## Recovery boundaries

- If the URL expires, inspect the browser and CLI state first. End only the
  failed login process you started, start one fresh native attempt, and reuse
  the same browser TaskSpace. Persistent failure needs its concrete cause fixed;
  do not create an unbounded login/refresh loop.
- The browser's localhost callback must reach the host running the CLI. On a
  remote/headless host, inspect the actual callback address. Use a verified SSH
  tunnel, or the installed CLI's supported `codex login --device-auth` flow when
  allowed; open its emitted URL/code flow in ego-browser as well. Never bypass
  workspace authentication policy to enable an unavailable method.
- Keep OAuth URLs, codes, credentials, and raw login logs out of git, PRs, and
  public reports. Report status and a masked account identity only.
- If ego-browser fails, diagnose its existing TaskSpace/installation using its
  skill. Do not silently substitute another browser for the user's requested one.
