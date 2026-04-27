# Bugbot Gate Rollout Handoff

This org-level `.github` repo now contains the shared reusable gate workflow at:

- `.github/workflows/bugbot-gate.yml`

Use the following files in each active application repository.

## 1) Thin per-repo caller workflow

Create `.github/workflows/bugbot-gate.yml` in each active repo:

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
    uses: ORG/.github/.github/workflows/bugbot-gate.yml@main
    with:
      sha: ${{ github.event.pull_request.head.sha }}
      bugbot_check_name: Cursor Bugbot
      timeout_minutes: 15
```

Replace `ORG` with the actual organization name.

## 2) Downstream expensive CI workflow

Create `.github/workflows/ci-after-gate.yml` in each active repo:

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
    name: Quality
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
      - uses: pnpm/action-setup@v4
        with:
          run_install: false
      - uses: actions/setup-node@v5
        with:
          node-version: lts/*
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck

  build:
    name: Build
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
      - uses: pnpm/action-setup@v4
        with:
          run_install: false
      - uses: actions/setup-node@v5
        with:
          node-version: lts/*
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

Important:

- Keep expensive CI out of direct `pull_request` triggers if the goal is to save Actions minutes.
- Use `github.event.workflow_run.head_sha` for checkout so downstream CI tests the same PR commit that passed the gate.

## 3) Org ruleset setup

1. Add a pilot repo with the two workflows above.
2. Open a real test PR and wait for checks to run.
3. In GitHub UI, verify the exact emitted check names.
4. Add required status checks in org rulesets using exact names.

Minimum required check:

- `Bugbot Gate`

Optional additional required checks (only after verified names exist):

- `Quality`
- `Build`
- `Playwright`

## 4) Rollout checklist (active repos first)

1. Keep shared gate logic centralized in `ORG/.github`.
2. Onboard one pilot active repo.
3. Confirm expensive CI does not start before gate success.
4. Confirm required-check names from real runs before locking rulesets.
5. Onboard the next 3-5 active repos.
6. Expand further only where there is active development.

## 5) v1 behavior and scope guardrails

`Bugbot Gate` v1 semantics:

- accept PR head SHA
- wait for `Cursor Bugbot` check (or configured check name)
- pass only when conclusion is `success`
- fail on non-success conclusion or timeout

Out of scope in v1:

- comment-resolution checks
- severity parsing
- stale-thread handling
- manual acknowledgment exceptions

## 6) Fork PR expectation (v1)

Fork handling is explicit in v1: support fork PRs only for non-secret downstream CI.

- `CI After Gate` runs from a `workflow_run` event, so use the `workflow_run` payload fields directly and do not assume `pull_request`-event semantics.
- `actions/checkout` with `github.event.workflow_run.head_sha` executes contributor-controlled code from the fork; treat that code as untrusted.
- Keep `CI After Gate` permissions minimal and avoid secrets-dependent steps unless you intentionally design and review a fork-safe model.
- Confirm `Cursor Bugbot` emits the named check for fork PRs in your org settings (including first-time contributor approval flows); if it does not, `Bugbot Gate` fails by design.
