# Main Account fallback repair

## Confirm both failure paths

Upstream OpenCodex 2.55.0 can strand CLI requests even when the dashboard selects
an eligible premium main with weekly headroom and no five-hour window:

1. **Existing binding:** Fill-first and Round-robin skip quota re-evaluation for
   a bound thread, so measured 100% usage can retain the exhausted account.
2. **Native caller bearer:** automatic Pool authentication excludes physical main
   when a CLI request carries its own ChatGPT token. Fixing affinity alone does
   not repair this path; an unbound request or quota retry can still miss main.

Trace `resolveCodexAccountForThreadDetailed`, `previewReusableAffinityAccount`,
`mayRebindAffinityForQuota`, and `resolveCodexAuthContext` in the installed source.
Correlate the failing CLI's actual account-labelled request, including HTTP 429.
Do not conclude there are no failures from logs filtered to bare `openai`.

## Versioned patch

The [bundled patch](opencodex-2.55.0-main-fallback.patch) targets
[`lidge-jun/opencodex` v2.55.0](https://github.com/lidge-jun/opencodex/tree/v2.55.0),
commit `1cc89cf88c39160e4bb1ee21f0fd37404dd6930b`. It includes two source changes,
regression tests, and the matching upstream documentation. Its source context is
covered by [OpenCodex's MIT license](OPENCODEX-LICENSE).

1. Discover the installed package and the process actually serving the configured
   port (`ocx status --json`). Common npm installations use
   `$(npm root -g)/@bitkyc08/opencodex`, but verify the executable's resolved target;
   another Node installation can own the running process.
2. Compare version **and pristine source bytes**. If the fix is already present,
   verify behavior and stop. For a different version or a modified source, inspect
   the current code and adapt the repair with regression tests; do not force this patch.
3. Use a clean checkout at the exact baseline. Read its `AGENTS.md`, including
   nested instructions for affected paths. With the real patch path:

   ```sh
   git apply --check /path/to/opencodex-2.55.0-main-fallback.patch
   git apply /path/to/opencodex-2.55.0-main-fallback.patch
   ```

4. Install its locked development dependencies and run the relevant routing,
   rotation, auth-context, Reserve, refresh, context-history, and compaction tests
   with the repository's test-home isolation. Run typecheck and required structure,
   privacy, and documentation checks. Use isolated test files/processes when Bun
   worker crashes or global mock contamination prevents a valid combined run;
   disclose any broader check that remains incomplete.
5. Archive the installed originals and patch under a new recoverable path.
   For the bundled patch, require installed `src/codex/routing.ts` and
   `src/codex/auth-context.ts` to match the pristine baseline before replacement.
   For an adapted repair, capture the installed modified files before editing,
   build and test on that baseline, and require the installation still matches
   those captured bytes before copying the tested adaptation. Preserve
   credentials and configuration. This Bun runtime loads TypeScript directly;
   only the two runtime source files require replacement for this version.
6. Save a recovery checkpoint, then use native `ocx restart`. It drains active
   requests and can return temporary 503s that Codex displays as model capacity.
   Poll `ocx ready --json` for a ready replacement PID. Avoid repeated restarts or
   broad process kills; restart only when activation requires it.
7. Verify the original CLI/Tracer conversation succeeds on its requested model
   and the request's attempt-level `accountLogLabel` is `main`. A small local Pool
   probe carrying a non-secret placeholder caller bearer plus account-id header
   can exercise the changed branch, but also inspect the original client's logs.

Rollback restores the two archived originals followed by the same native restart.
An npm/package upgrade can replace these local edits; recheck source and behavior
before reapplying anything. Do not claim this patch is an upstream release.

## Invariants to retain when adapting

- Known 100% releases Fill-first/Round-robin affinity only toward an eligible,
  genuinely cooler account with headroom; 99% retains the binding. Unknown quota
  is not zero. `autoSwitchThreshold: 0` retains its existing proactive-switch semantics.
- Only **automatic Pool + loopback admission + no exact selector + non-Reserve**
  includes stored main despite a native caller token. All shared callers, including
  initial requests, quota retry, and compaction, use the central auth resolver.
- Direct, remote, exact-account, Reserve, model eligibility, hard-lock, and
  profile-drain/claim checks remain intact. No caller bearer is persisted as main.
- Preserve the validated caller fallback when no stored account exists, an
  eligible stored main needs reauthentication, or its token disappears. Preserve
  the cooled-account fallback and its same-account protection. Abort and an
  excluded `MAIN_CODEX_ACCOUNT_ID` must not trigger that credential again.

Useful focused tests in the upstream checkout:

```text
tests/codex-integration/codex-routing.test.ts
tests/codex-integration/codex-pool-rotation.test.ts
tests/codex-integration/codex-auth-context.test.ts
tests/codex-integration/reserve-auth-context.test.ts
tests/responses/responses-pool-401-refresh.test.ts
tests/responses/responses-native-main-refresh.test.ts
tests/server/context-history.test.ts
tests/responses/responses-compaction-routing.test.ts
```
