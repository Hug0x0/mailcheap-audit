# 02 — Dependencies

Method: `npm audit --package-lock-only` against the committed `package-lock.json` (no install, no scripts executed). Run on 2026-10-22.

## Result

| Severity | Count |
|---|---:|
| Critical | 0 |
| High | 9 |
| Moderate | 1 |
| Low | 0 |

### Runtime (production) dependencies

| ID | Package | Installed | Advisory | Assessment |
|---|---|---|---|---|
| DEP-01 | `sanitize-html` | 2.18.0 | [GHSA-c9rx-x947-fp3h](https://github.com/advisories/GHSA-c9rx-x947-fp3h) — unbounded recursion when re-sanitizing the fallback of a discarded `<iframe>` (stack overflow, `nestingLimit` not enforced) | **Low.** `lib/safe-html.js` does not allow `iframe`, and `lib/clean-html.js` strips `iframe` elements *with their content* before content is stored, which removes the trigger in the normal path. Still a direct, runtime dependency processing user HTML: bump to `>=2.18.1` (non-breaking). |

### Development-only dependencies

All 9 "high" findings come from the `braces` ReDoS/stack-exhaustion advisory ([GHSA-vfj7-8cjw-p6xm](https://github.com/advisories/GHSA-vfj7-8cjw-p6xm)) pulled in transitively through `fast-glob` → `micromatch` → `braces`, by:

- `shadcn` (CLI used to scaffold UI components) via `@shadcn/registry` / `ts-morph`
- `eslint-config-next` via `@next/eslint-plugin-next`

| ID | Package chain | Assessment |
|---|---|---|
| DEP-02 | `shadcn` → … → `braces@3.0.3` | **Info.** Dev tooling only, never shipped, inputs are developer-controlled. `npm audit`'s suggested "fix" is a semver-major *downgrade* (`shadcn@1.0.0`, `eslint-config-next@14`) — do **not** apply it. Wait for upstream bumps or add an `overrides` entry once a patched `braces` exists. |
| DEP-03 | `eslint-config-next` → … → `braces@3.0.3` | **Info.** Same as above. |

## Versioning observations

- Dependencies use caret ranges (`^`) except `next`, `react`, `react-dom`, `eslint-config-next`, which are pinned — sensible for the framework.
- The lockfile is committed, so builds are reproducible.
- No automated dependency update tool (Dependabot/Renovate) is configured; with no CI either, advisories like DEP-01 rely on manual checks. **Recommendation:** add `.github/dependabot.yml` and an `npm audit --omit=dev` step to CI.
