# PR Checks Audit

Current state of PR workflows, branch protection, and repository settings across active kyndig repositories.

Last updated: 2026-05-11

---

## Repository Settings Overview

| Repo | Default Branch | Delete Branch on Merge | main Protected | Required Status Checks |
|------|---------------|----------------------|----------------|------------------------|
| TF | feat/foundation | ❌ | ❌ | — |
| ritz | main | ✅ | ❌ | — |
| kynd-web | main | ✅ | ✅ | `Check` |
| kynd-web-new | main | ✅ | ✅ | `Quality`, `Build`, `Playwright` |
| switchto | main | ❌ | ❌ | — |
| varde | main | ✅ | ✅ | `PR Metadata Check`, `SDK tests (PR, fast)`, `Check Bugbot Comments` |
| varde-web | main | ❌ | ❌ | — |
| yr-wfc | main | ✅ | ❌ | — |
| brreg-search | main | ✅ | ❌ | — |
| faen-ta | main | ❌ | ❌ | — |
| kynd-bid-system | main | ✅ | ❌ | — |
| spork | main | ✅ | ❌ | — |
| spork-web | main | ✅ | ❌ | — |
| triager | main | ❌ | ❌ | — |

### Gaps to close

**Branch deletion after merge** — must be enabled on: `TF`, `switchto`, `varde-web`, `faen-ta`

**Main branch protection** — must be added on: `TF`, `ritz`, `switchto`, `varde-web`, `yr-wfc`, `brreg-search`, `faen-ta`, `kynd-bid-system`, `spork`, `spork-web`

**Default branch normalisation** — `TF` currently uses `feat/foundation` as default branch; align to `main` before enforcing org-wide `main` rulesets.

**Script vocabulary drift** — local quality/lint/type scripts differ by repo (`check`, `verify`, `type-check`, `typecheck`). Standardise on `docs/script-contract.md` during onboarding PRs.

**Protected repos to normalise** — `kynd-web` uses `Check` (stale name from an old local CI job), `kynd-web-new` and `varde` use custom local check names. All three need to converge to the shared `Bugbot Gate` baseline once onboarded.

---

## PR Workflows Per Repo

### ritz (Swift/macOS)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `Bugbot Review Check` | pull_request, review events | Waits for Cursor Bugbot check completion; GraphQL unresolved-thread check (all cycles, no cycle-boundary logic) |
| `PR Validation` | pull_request | SwiftLint strict (macOS runner), PR metadata (issue link, ContractImpact, changelog warning), failure comment on PR |
| `Integration Tests` | schedule (weekly) | Not on PR |
| `Nightly Tests` | schedule | Not on PR |
| `Plan Sync`, `Planning JSON`, `Project Status Sync` | Not PR-specific | |

**Bugbot first**: No — SwiftLint runs in parallel with Bugbot.
**Expensive CI on `pull_request`**: Yes (SwiftLint on macOS runner).
**Shared gate**: Not yet onboarded.

---

### kynd-web (pnpm web app)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `CI` → job `Check` | pull_request, push (non-main) | pnpm install, `pnpm check`, `pnpm build` |
| `Issue Hygiene` | Not PR-specific | |

**Bugbot first**: No.
**Expensive CI on `pull_request`**: Yes.
**Shared gate**: Not yet onboarded.

---

### kynd-web-new (pnpm web app)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `CI` → jobs `Quality`, `Build`, `Playwright` | pull_request (main), push (main) | All three jobs run directly on pull_request |
| `Lighthouse Budget` | schedule (Monday), manual | Not on PR |

**Bugbot first**: No — all three expensive jobs run in parallel.
**Expensive CI on `pull_request`**: Yes.
**Shared gate**: Not yet onboarded.

Note: `kynd-web-new` already has the canonical `Quality`, `Build`, `Playwright` job name split; it is the best pilot for moving expensive CI behind the gate.

---

### switchto

No workflow files found. No protection. Branch deletion disabled.

---

### TF (SvelteKit/npm web app)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| None | — | No PR checks configured. `package.json` exposes `npm run check` and `npm run build`. |

**Bugbot first**: No (no gate configured).
**Expensive CI on `pull_request`**: No (no PR CI configured).
**Shared gate**: Not yet onboarded.

Recommended target checks after onboarding:

- `Bugbot Gate`
- `CI After Gate / Quality` (`npm run check`)
- `CI After Gate / Build` (`npm run build`)

---

### varde (Python SDK / workflow kit)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `Bugbot Review Check (Kit)` | pull_request, review events | Configurable reviewer/body matchers; current-cycle GraphQL thread check (most advanced local implementation) |
| `PR Validation (Kit)` | pull_request | Issue link / WorkDoc, ContractImpact, release field, SDK docs drift, work registry validation |
| `Tests (Kit)` → `SDK tests (PR, fast)` | pull_request (path-filtered) | Fast SDK tests, no network; nightly lane with network smoke |
| Various project sync, release, staleness, epic workflows | Not PR-specific | |

**Bugbot first**: No — SDK tests and metadata check run in parallel with Bugbot.
**Expensive CI on `pull_request`**: SDK tests are relatively cheap and path-filtered.
**Shared gate**: Not yet onboarded. The local Kit implementations are the reference for the shared Bugbot gate upgrade.

---

### varde-web

No workflow files found. No protection. Branch deletion disabled.

---

### yr-wfc (Raycast extension)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `Raycast CI` | pull_request, push (main) | Metadata validation, migration drift, npm ci, build, typecheck, lint, unit tests; macOS runner |

**Bugbot first**: No.
**Expensive CI on `pull_request`**: Yes (macOS runner).
**Shared gate**: Not yet onboarded.

---

### brreg-search (Raycast extension)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `Raycast CI` | pull_request, push (main) | Metadata validation, migration drift, npm ci, lint, build; macOS runner |

**Bugbot first**: No.
**Expensive CI on `pull_request`**: Yes (macOS runner).
**Shared gate**: Not yet onboarded.
**Divergence from yr-wfc**: Different Node version, action versions, no type check or unit tests.

---

### faen-ta

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `substack-rebuild` | Not PR-specific | Deploy/content workflow |

No PR checks. No protection. Branch deletion disabled.

---

### kynd-bid-system (pnpm monorepo / Turborepo)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `CI` → `validate` + `bugbot-gate` | pull_request, push (main) | `validate`: turbo lint, typecheck, test; `bugbot-gate`: inline GraphQL thread check only (no check-run wait, no cycle boundary) |

**Bugbot first**: No — `validate` and `bugbot-gate` run in parallel.
**Expensive CI on `pull_request`**: Yes.
**Shared gate**: Partial inline implementation, no check-run wait, no cycle-boundary semantics.

---

### spork (CLI tool, shell/macOS)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `CI` | pull_request, push (main/master) | `make ci` (bats + shellcheck); macOS runner |

**Bugbot first**: No.
**Expensive CI on `pull_request`**: Yes (macOS runner).
**Shared gate**: Not yet onboarded.

---

### spork-web (pnpm + Python)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `CI` | pull_request, push (main) | pnpm + Python setup, contract generation/check, lint, typecheck, test, test:api, build |

**Bugbot first**: No.
**Expensive CI on `pull_request`**: Yes.
**Shared gate**: Not yet onboarded.

---

## Existing Bugbot Implementations

Three diverging local Bugbot-comment check implementations exist. All should be replaced by the shared reusable `Bugbot Gate`.

| Repo | Waits for check run? | Cycle-boundary logic? | Configurable matchers? |
|------|---------------------|----------------------|----------------------|
| ritz | ✅ (5 min timeout, any completion) | ❌ | ❌ |
| varde | ❌ (no check-run wait) | ✅ | ✅ |
| kynd-bid-system | ❌ | ❌ | ❌ |

The shared gate combines check-run wait (from `ritz`) with cycle-boundary logic and configurable matchers (from `varde`).

---

## Repo Categories

| Category | Repos |
|----------|-------|
| Web app, pnpm | `kynd-web`, `kynd-web-new`, `spork-web` |
| Web app, npm (SvelteKit) | `TF` |
| Node/pnpm monorepo | `kynd-bid-system` |
| Raycast extension | `yr-wfc`, `brreg-search` |
| Swift/macOS app | `ritz`, `triager` |
| Python SDK / kit | `varde` |
| CLI tool (shell/macOS) | `spork` |
| No known CI yet | `switchto`, `varde-web`, `faen-ta` |

### triager (Swift/macOS — onboarding)

| Workflow | Trigger | Notes |
|----------|---------|-------|
| `Bugbot Gate` | pull_request | Shared reusable gate; waits for Cursor Bugbot + unresolved thread check |
| `CI After Gate` | workflow_run | SPM tests + Xcode app build via shared `swift-ci.yml`; SwiftLint deferred |

**Bugbot first**: Yes — expensive macOS CI runs only after gate passes.
**Expensive CI on `pull_request`**: No (gated via `workflow_run`).
**Shared gate**: Onboarded (pending merge of `swift-ci.yml` to `kyndig/.github@main` and first push to remote).
