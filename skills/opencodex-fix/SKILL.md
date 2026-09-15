---
name: opencodex-fix
description: Use when OpenCodex keeps Codex CLI or Tracer on usage-limited accounts despite an eligible Main Account, or when Codex is signed out and needs browser login through ego-browser.
---

# OpenCodex fix

Restore usable authentication and verify which account actually serves the failing
client. A dashboard selection, browser session, or installed patch alone is insufficient.

## Establish the failing layer

Run these read-only checks in the failing client's environment:

```sh
codex login status
ocx ready --json
ocx status --json
ocx account list openai --json
ocx account current openai --json
```

Use installed help if commands differ. Respect its effective Codex home and
credential store; missing `auth.json` does not prove logout because keyring auth
may be active. Inspect only needed fields; never print credentials.

| Evidence | Next action |
|---|---|
| CLI reports signed out or unusable authentication | Follow [login with ego-browser](references/ego-login.md): native `codex login`, or Pool account re-authentication when `codexAccountMode` is `pool`. A revoked session strands every Pool account that shares it — re-authenticate each, not just the main. |
| Added accounts have known short-window 100%; main has known weekly headroom and no short window | Follow [Main Account routing repair](references/main-account-repair.md). |
| `Selected model is at capacity` / `server_overloaded` | Compare its timestamp with proxy restart/drain and upstream logs. Wait for readiness after drain; distinguish persistent upstream overload. This message can also mask a `token_revoked` / `refresh_token_invalidated` 401 — check the attempt's real upstream status before concluding capacity. |
| Dashboard looks healthy but CLI fails | Correlate the actual CLI request and its serving account before changing configuration. |

A weekly-only account is not missing all quota information. Score its known
windows; do not invent a five-hour limit or assume unlimited weekly allowance.
Completely unknown quota is not evidence of available headroom.

Read request logs across **all account-labelled providers**: `provider=openai`
can hide failures recorded under `openai-<account-label>`. Inspect attempt-level
`accountLogLabel`, terminal outcome, model, and request/conversation identifiers.
Use the existing authorized management interface; keep its secrets out of output.

## Scope and completion

- Preserve unrelated settings and accounts. Archive replaced files recoverably.
- A usage-limit error alone is not a reason to log out, erase authentication, or
  initiate browser login. Request-only bearer tokens must never be persisted as main.
- For the versioned repair, retain Direct, remote, explicit-account, Reserve,
  hard-lock, drain, and failed-account retry boundaries described in the reference.
- Before a required restart, save the patch, rollback location, and verification
  checkpoint. Drain can interrupt the agent's own CLI; resume from that checkpoint.
- Require a completed minimal request **and the original client's serving-account
  log** after recovery. Keep the requested model unless the user authorizes a change.
  One minimal request from that original client can satisfy both checks.
- When a revoked session stranded multiple Pool accounts, restore and verify
  **each** account the fix is scoped to, not only the active main; disclose any
  seat left unusable rather than removing it. Re-authenticate one flow at a time.
- Report what changed, runtime readiness, account/result evidence, and any check
  that could not finish. Distinguish a local patch from an upstream release.

If user interaction is needed for password, MFA, passkey, or account selection,
hand control over in ego-browser and resume the same login flow afterwards.
