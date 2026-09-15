# Native Codex login through ego-browser

Use only when the effective CLI reports missing/unusable login, or the user
explicitly requests an account change. A logged-in ChatGPT browser session does
not establish CLI authentication. These steps follow the
[official Codex authentication documentation](https://learn.chatgpt.com/docs/auth).

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
