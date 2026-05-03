# Org Repository Baseline

Required settings and branch protection configuration for all active kyndig repositories.
Apply these before or alongside the shared workflow rollout.

---

## Repository Settings (apply to every active repo)

### Automatic branch deletion after merge

Enable `delete_branch_on_merge` on every repo. This prevents stale branches accumulating after PRs are merged.

**Currently missing on:** `switchto`, `varde-web`, `faen-ta`

Apply via GitHub UI (Settings → General → Pull Requests → Automatically delete head branches)
or via the API / Terraform if you manage settings programmatically.

### Default branch

All active repos already use `main` as the default branch.

---

## Main Branch Protection (apply to every active repo)

Minimum ruleset or branch protection configuration for `main`:

| Setting | Value | Reason |
|---------|-------|--------|
| Require a pull request before merging | ✅ | Core policy: no direct pushes to main |
| Require approvals | 0 (or 1 if preferred) | Gate is the quality signal; reviews are optional |
| Dismiss stale reviews when new commits are pushed | ✅ recommended | Prevents stale approvals from carrying through |
| Require status checks to pass before merging | ✅ | See required checks below |
| Require branches to be up to date before merging | ✅ | Prevents merging stale branches that bypass gate |
| Do not allow bypassing the above settings | ✅ | Enforce admins too |

**Currently unprotected:** `ritz`, `switchto`, `varde-web`, `yr-wfc`, `brreg-search`, `faen-ta`, `kynd-bid-system`, `spork`, `spork-web`

**Currently protected but needing normalisation:** `kynd-web` (requires `Check` — stale name), `kynd-web-new` (no `Bugbot Gate`), `varde` (no `Bugbot Gate`)

---

## Required Status Checks

Add required checks in the following sequence to avoid blocking PRs before workflows exist.

### Step 1 — Available on all onboarded repos immediately

After adding the thin `bugbot-gate.yml` caller and the `CI After Gate` workflow to a repo, require:

- `Bugbot Gate`

This is the minimum gate. Do not add any other required check until you have observed its exact emitted name in a real PR.

### Step 2 — Add downstream checks after verification

Once you have confirmed exact emitted check names from a real PR run, add the relevant downstream checks. Example names by category (verify before locking):

| Category | Expected check names |
|----------|---------------------|
| Web pnpm | `CI After Gate / Quality`, `CI After Gate / Build` |
| Web pnpm + Playwright | `CI After Gate / Quality`, `CI After Gate / Build`, `CI After Gate / Playwright` |
| Raycast extension | `CI After Gate / Raycast CI` |
| Python SDK | `CI After Gate / SDK Tests` |
| Swift/macOS | `CI After Gate / SwiftLint` |
| CLI tool | `CI After Gate / CI` |

Do not guess check names. The check name in the GitHub UI is what to use.

### Current protected repo gaps

| Repo | Current required checks | Action needed |
|------|------------------------|---------------|
| kynd-web | `Check` | Replace with `Bugbot Gate` + category checks after onboarding |
| kynd-web-new | `Quality`, `Build`, `Playwright` | Add `Bugbot Gate`; migrate expensive CI to `CI After Gate` |
| varde | `PR Metadata Check`, `SDK tests (PR, fast)`, `Check Bugbot Comments` | Add `Bugbot Gate`; migrate to shared gate + `CI After Gate` |

---

## Org-Level Ruleset (recommended)

Rather than configuring each repo independently, consider a single org-level ruleset targeting all active repos. This avoids per-repo drift.

Recommended org ruleset:

```
Target: all repositories (or specific repos by name pattern)
Target branch: main

Rules:
  - Require pull request
  - Required status checks:
      - Bugbot Gate
  - Require branches to be up to date
  - Delete head branch on merge (enforced at ruleset level)
```

Per-repo downstream checks (`Quality`, `Build`, etc.) cannot be required org-wide because they differ per category. Add them as repo-level required checks after verifying names.

---

## Rollout Sequence

Apply in this order to avoid breaking existing PRs:

1. **Enable branch deletion** on `switchto`, `varde-web`, `faen-ta` (no workflow dependency, safe to do immediately).
2. **Add main protection** to all unprotected repos with PR-required-only (no required status checks yet).
3. **Onboard pilot repo** with shared gate workflows (recommended: `kynd-web-new`).
4. **Verify check names** from a real pilot PR.
5. **Add `Bugbot Gate` as required check** on pilot repo.
6. **Expand to next 3–5 active repos** following the same pattern.
7. **Migrate existing protected repos** (`kynd-web`, `kynd-web-new`, `varde`) to shared gate after their local implementations are removed.
8. **Do not force rollout** to inactive repos (`switchto`, `varde-web`, `faen-ta`) until they have active development.

---

## Normalising Currently Protected Repos

### kynd-web

1. Add `bugbot-gate.yml` caller and `ci-after-gate.yml` (pnpm web pattern).
2. Remove direct `pull_request` trigger from the existing `ci.yml`.
3. Verify `Bugbot Gate`, `CI After Gate / Quality`, `CI After Gate / Build` appear in checks.
4. Update branch protection: replace `Check` with `Bugbot Gate` + verified downstream names.

### kynd-web-new

1. Add `bugbot-gate.yml` caller.
2. Replace the direct `pull_request` trigger in `ci.yml` with a `ci-after-gate.yml` using `workflow_run`.
3. Verify check names (they may change from `Quality` to `CI After Gate / Quality`).
4. Update branch protection: add `Bugbot Gate`, update downstream check names.

### varde

1. Add `bugbot-gate.yml` caller.
2. Replace local `bugbot-check.yml` with the shared gate (remove local after migration).
3. Move SDK tests to `ci-after-gate.yml` triggered by `workflow_run`.
4. Keep `pr-validation.yml` on `pull_request` (it is cheap and not blocked by the gate).
5. Update branch protection: replace `Check Bugbot Comments` with `Bugbot Gate`; update test check name if it changes.
