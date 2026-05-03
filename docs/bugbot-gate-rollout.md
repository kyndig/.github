# Bugbot Gate Rollout Handoff

The shared org-level gate lives in this repo at:

- `.github/workflows/bugbot-gate.yml` — reusable gate (wait for Cursor Bugbot + unresolved thread check)
- `.github/workflows/pr-metadata-check.yml` — reusable PR metadata/description check
- `.github/workflows/node-pnpm-quality.yml` — reusable lint + typecheck
- `.github/workflows/node-pnpm-build.yml` — reusable build
- `.github/workflows/node-pnpm-playwright.yml` — reusable Playwright
- `.github/workflows/raycast-ci.yml` — reusable Raycast extension CI
- `.github/workflows/python-sdk-tests.yml` — reusable Python SDK tests

Each active application repo needs two workflow files and nothing else for the baseline.

---

## 1) Thin Bugbot Gate caller (all repos)

Create `.github/workflows/bugbot-gate.yml` in each active repo.
Replace `kyndig` with the actual organisation name if it ever changes.

```yaml
name: Bugbot Gate

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  checks: read
  pull-requests: read

jobs:
  gate:
    name: Bugbot Gate
    uses: kyndig/.github/.github/workflows/bugbot-gate.yml@main
    with:
      sha: ${{ github.event.pull_request.head.sha }}
      pull_number: ${{ github.event.pull_request.number }}
      bugbot_check_name: Cursor Bugbot
      timeout_minutes: 15
```

---

## 2) CI After Gate — per repo category

The expensive CI workflow uses `workflow_run` so it only starts after `Bugbot Gate` succeeds.
Use `github.event.workflow_run.head_sha` for checkout so tests always run on the exact commit that passed the gate.

### Web app (pnpm) — kynd-web, kynd-web-new, spork-web

Create `.github/workflows/ci-after-gate.yml`:

```yaml
name: CI After Gate

on:
  workflow_run:
    workflows: ["Bugbot Gate"]
    types: [completed]

permissions:
  contents: read

jobs:
  quality:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: kyndig/.github/.github/workflows/node-pnpm-quality.yml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      lint_command: pnpm lint
      typecheck_command: pnpm typecheck
      # format_command: pnpm format   # uncomment if the repo enforces formatting

  build:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: kyndig/.github/.github/workflows/node-pnpm-build.yml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}

  playwright:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: kyndig/.github/.github/workflows/node-pnpm-playwright.yml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      test_command: pnpm exec playwright test
      # Remove or adjust this job if the repo has no Playwright tests.
```

**kynd-web** uses `pnpm check` instead of separate lint/typecheck — override accordingly:

```yaml
  quality:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: kyndig/.github/.github/workflows/node-pnpm-quality.yml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      lint_command: pnpm check
      typecheck_command: "true"   # check already includes typecheck; skip explicit step
```

**spork-web** also runs Python API tests — add a bespoke step or a separate job for `pnpm test:api` in a local `ci-after-gate.yml` that extends the shared quality/build jobs.

---

### Node/pnpm monorepo (Turborepo) — kynd-bid-system

```yaml
name: CI After Gate

on:
  workflow_run:
    workflows: ["Bugbot Gate"]
    types: [completed]

permissions:
  contents: read

jobs:
  quality:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: kyndig/.github/.github/workflows/node-pnpm-quality.yml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      lint_command: pnpm turbo lint
      typecheck_command: pnpm turbo typecheck

  test:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: kyndig/.github/.github/workflows/node-pnpm-build.yml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      build_command: pnpm turbo test
```

---

### Raycast extension — yr-wfc, brreg-search

```yaml
name: CI After Gate

on:
  workflow_run:
    workflows: ["Bugbot Gate"]
    types: [completed]

permissions:
  contents: read

jobs:
  raycast:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: kyndig/.github/.github/workflows/raycast-ci.yml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      node_version: "22"
      raycast_migration_version: "1.103.0"
      # yr-wfc has unit tests; brreg-search currently does not:
      # test_command: npm run test:unit
```

---

### Python SDK — varde

```yaml
name: CI After Gate

on:
  workflow_run:
    workflows: ["Bugbot Gate"]
    types: [completed]

permissions:
  contents: read

jobs:
  sdk-tests:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: kyndig/.github/.github/workflows/python-sdk-tests.yml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      python_version: "3.11"
      working_directory: sdk/adapters/terminal
      test_command: python -m unittest discover -s tests
      skip_network_tests: true
      upload_artifacts: true
      artifact_name: varde-test-output
```

---

### Swift/macOS — ritz

Ritz uses SwiftLint on a macOS runner. There is no shared reusable workflow for Swift yet. Create a local `ci-after-gate.yml` that calls SwiftLint directly after the gate:

```yaml
name: CI After Gate

on:
  workflow_run:
    workflows: ["Bugbot Gate"]
    types: [completed]

permissions:
  contents: read

jobs:
  swiftlint:
    name: SwiftLint
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: macos-15
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
      - name: Install SwiftLint
        run: brew install swiftlint
      - name: Run SwiftLint
        run: swiftlint lint --strict --reporter github-actions-logging
```

---

### CLI tool — spork

Spork uses bats and shellcheck on macOS. Create a local `ci-after-gate.yml`:

```yaml
name: CI After Gate

on:
  workflow_run:
    workflows: ["Bugbot Gate"]
    types: [completed]

permissions:
  contents: read

jobs:
  ci:
    name: CI
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: macos-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
      - name: Install dependencies
        run: brew install bats-core shellcheck
      - name: Run CI
        run: make ci
```

---

## 3) Optional PR metadata check

If the repo enforces PR description requirements, also create `.github/workflows/pr-metadata.yml`:

```yaml
name: PR Validation

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, edited]

permissions:
  contents: read
  pull-requests: read
  issues: read

jobs:
  metadata:
    name: PR Metadata Check
    uses: kyndig/.github/.github/workflows/pr-metadata-check.yml@main
    with:
      require_issue_ref: true
      allow_work_doc: false            # set true for repos using WorkDoc: docs/work/<slug>.md
      require_contract_impact: false   # set true for repos with a docs/contracts/ directory
```

---

## 4) Org ruleset setup

1. Add the two per-repo workflow files above to a pilot repo.
2. Open a real test PR and wait for both workflows to run.
3. In GitHub UI → repo → Actions, confirm the exact emitted check names.
4. Add required status checks in org or repo rulesets using the exact names.

Minimum required check for every repo:

- `Bugbot Gate`

Downstream checks to add once check names are verified (example names below — confirm from a real PR before locking):

- Web pnpm: `CI After Gate / Quality`, `CI After Gate / Build`, `CI After Gate / Playwright`
- Raycast: `CI After Gate / Raycast CI`
- Python SDK: `CI After Gate / SDK Tests`

Do not guess check names and lock them into policy without observing them first.

---

## 5) Rollout checklist

1. Enable base repo settings where missing: branch deletion after merge, `main` protection, require PRs.
2. Implement shared workflows in this `.github` repo (done).
3. Onboard one pilot repo (recommended: `kynd-web-new` — already has clean job categories).
4. Confirm `Bugbot Gate` appears as a required check from a real PR.
5. Confirm expensive CI does **not** start before `Bugbot Gate` succeeds.
6. Lock required checks via ruleset.
7. Remove or disable local Bugbot-comment check in the onboarded repo.
8. Onboard next 3–5 active repos (see `docs/pr-checks-audit.md` for priority order).
9. Do not force rollout to inactive repos.

---

## 6) Gate semantics

`Bugbot Gate` passes only when both conditions are true:

- The `Cursor Bugbot` check run has completed (any conclusion — completion is the trigger).
- No unresolved Bugbot review threads exist for the **current push cycle** (threads posted after the most recent commit or force-push to the PR).

Current-cycle scoping means resolving old threads from a previous commit does not re-open the gate; only threads from the latest push count.

Out of scope in v1:

- Severity-level filtering (Low / Medium / High / Critical treated equally)
- Manual acknowledgment exceptions
- Stale-thread dismissal

---

## 7) Fork PR expectation

Fork PRs are supported as long as:

- `CI After Gate` steps do not depend on repo secrets.
- `actions/checkout` uses `github.event.workflow_run.head_sha` (contributor-controlled code — treat as untrusted for secrets-sensitive steps).
- `Cursor Bugbot` emits the named check for fork PRs in your org settings. If first-time contributor approval is required by GitHub, the gate will not find the check and will time out by design.
