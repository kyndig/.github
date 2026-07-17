# Shared Script Contract

This document defines a shared local and CI script vocabulary for active kyndig repositories.

It exists to remove script-name drift across stacks (web, monorepo, Raycast, Swift), so common tasks like `check` and `check:fix` have predictable meaning.

---

## Scope

### What this repo can centralize

- Script naming contract and semantics.
- Reusable CI workflow defaults and command inputs.
- Copy-paste template script blocks for common stack categories.

### What this repo cannot centralize

- Runtime inheritance of `package.json` scripts into other repos.
- Forced package-manager unification across stacks.
- Swift/Xcode workflows into npm scripts.

Per-repo `package.json` remains the implementation layer. This repo is the policy and contract layer.

---

## Canonical Script Vocabulary

Use these names for Node-based repositories.

| Purpose | Canonical script | Semantics |
|---------|------------------|-----------|
| Aggregate quality | `check` | Non-mutating quality gate (lint + typecheck + format check + stack extras). |
| Aggregate autofix | `check:fix` | Safe write-mode lint/format fixes. No claim to fix type errors. |
| Lint | `lint` | Read-only lint run. |
| Lint autofix | `lint:fix` | Write-mode lint run. |
| Type checking | `typecheck` | Canonical spelling: no hyphen. |
| Format check | `format:check` | Read-only formatter validation. |
| Format write | `format:write` | Write-mode formatting. |
| Build | `build` | Production/build export command. |
| Tests | `test` | Default unit/integration test entrypoint. |
| E2E tests (optional) | `test:e2e` | End-to-end test entrypoint (for example Playwright). |

### Contract rules

1. `check` is reserved for the aggregate quality gate.
2. Stack-specific checks must be namespaced (`svelte:check`, `contracts:check`) and composed into `check`.
3. Legacy aliases may exist during migration (`type-check`, `verify`, `fix-lint`), but canonical names must also exist.
4. Package managers stay stack-native: same script names, different invocation (`pnpm`, `npm run`).

---

## CI Mapping

Reusable workflows should map to the same local script contract.

| Workflow | Input | Expected contract usage |
|----------|-------|-------------------------|
| `.github/workflows/node-pnpm-quality.yml` | `check_command` (optional) | If set, run one aggregate quality command and skip atomic steps. |
| | `lint_command` | Usually `pnpm lint`. |
| | `typecheck_command` | Usually `pnpm typecheck`. |
| | `format_command` | Usually empty or `pnpm format:check`. |
| `.github/workflows/node-pnpm-build.yml` | `build_command` | Usually `pnpm build`. |
| `.github/workflows/node-pnpm-playwright.yml` | `test_command` | Prefer command backing local `test:e2e`. |
| `.github/workflows/raycast-ci.yml` | `build_command` | Prefer `npm run build`. |
| | `lint_command` | Prefer `npm run lint`. |
| | `typecheck_command` | Prefer `npm run typecheck`. |
| | `test_command` | Prefer `npm run test` when tests are present. |

### Why `check_command` exists

Some repos currently expose only an aggregate `check` script. `check_command` avoids anti-pattern callers such as:

- `lint_command: pnpm check`
- `typecheck_command: "true"`

Use:

```yaml
check_command: pnpm check
```

---

## Stack Mappings

### Web app (pnpm)

Target scripts:

- `check`, `check:fix`
- `lint`, `lint:fix`
- `typecheck`
- `format:check`, `format:write`
- `build`, `test`, optional `test:e2e`

### Web app (npm, SvelteKit / TF)

Target scripts:

- Keep `svelte:check` for raw Svelte checks.
- Define contract `check` as aggregate entrypoint (initially may call only `svelte:check`).
- Add remaining canonical scripts as tooling is introduced.

### Raycast extension (npm)

Target scripts:

- Canonical names above.
- Script bodies can wrap Raycast CLI (`ray build`, `ray lint`) while preserving canonical names.
- Keep temporary aliases (`type-check`, `format-check`) until consumers migrate.

### Node monorepo (pnpm/turbo)

Target scripts:

- Canonical names at root, composed via `turbo run ...`.
- Keep package-local names where needed; root scripts provide stable developer entrypoints.

### Swift/macOS app

No npm script contract requirement. Use a thin local adapter such as:

- `make check`
- `scripts/check.sh`

---

## Migration Checklist (per repo)

1. Add missing canonical scripts (non-breaking aliases first).
2. Keep legacy aliases temporarily to avoid abrupt local breakage.
3. Update `ci-after-gate.yml` caller to use canonical command defaults (or `check_command`).
4. Validate emitted GitHub check names are unchanged where required by branch protection.
5. Remove deprecated aliases in a follow-up PR after team adoption.

---

## Forbidden Overloads

Do not use `check` for:

- Svelte-only checks without the aggregate quality meaning.
- Domain-specific checks like `contracts:check`.
- Install wrappers (`npm install && ...`) that blur task intent.

Use namespaced scripts for specialized checks and compose them into `check` when they are part of quality gating.
