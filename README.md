# opencodex-fix

A Codex skill for recovering OpenCodex account routing and completing native
Codex login through **ego-browser**.

- Restore eligible Main Account selection when added accounts hit their five-hour limits.
- Repair exhausted Fill-first/Round-robin thread bindings and local CLI bearer routing.
- When Codex is signed out, complete its own OAuth flow in ego-browser and verify the CLI callback.

## Install

```sh
git clone https://github.com/deepnmr/opencodex-fix.git
mkdir -p ~/.codex/skills
ln -s "$(pwd)/opencodex-fix/skills/opencodex-fix" ~/.codex/skills/opencodex-fix
```

If that destination already exists, inspect it before changing it. The link keeps
the installed skill synchronized with this checkout. The `ego-browser` CLI and
skill must be installed separately for browser login.

Invoke `$opencodex-fix`, or ask to fix OpenCodex usage-limit routing or restore
Codex login. Read the [skill](skills/opencodex-fix/SKILL.md) for scope and checks.

The bundled patch targets upstream OpenCodex **2.55.0** only. It includes regression
tests and documentation, and is not automatically applied to other versions.
Package upgrades may overwrite a local runtime patch.

The skill is MIT licensed. The patch includes OpenCodex-derived context covered
by the [upstream MIT notice](skills/opencodex-fix/references/OPENCODEX-LICENSE).
