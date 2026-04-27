# Instructions for Another LLM: Centralized GitHub Gate

You are implementing an organization-level GitHub Actions gate that prevents expensive CI from running until a cheap Bugbot gate has passed.

Do **not** assume this workspace contains all target repositories.
Do **not** ask the user which repo to plan for unless a required detail is genuinely missing.
Assume the desired outcome is a multi-repo design with:

- one shared implementation in the org-level automation repo
- thin per-repo caller workflows in active repositories
- expensive CI moved behind the gate
- future compatibility with a GitHub App

Your job is to either:

1. implement the shared pieces in the currently available shared repo, and clearly describe the required per-repo follow-up changes, or
2. if asked for documentation only, produce the plan and files exactly as specified below.

## What the user wants

The user wants a centralized GitHub gate with these properties:

- expensive CI should not run before the gate passes
- the gate should initially depend only on Cursor Bugbot succeeding
- the design should stay DRY across many repositories
- only the currently active repositories need onboarding first
- a future GitHub App may replace the gate implementation later

## Important constraints

- Do not over-engineer v1.
- Do not start with comment-resolution semantics, severity parsing, or stale-thread logic.
- Do not propose a GitHub App as the immediate solution.
- Do not keep expensive CI directly on `pull_request` if the goal is to save Actions minutes.
- Do not force rollout to all repositories at once.

## The intended architecture

Use two workflow layers:

1. `Bugbot Gate`
2. `CI After Gate`

### `Bugbot Gate`

A cheap workflow that:

- runs on PR events
- waits for the `Cursor Bugbot` check on the PR head SHA
- passes only if Bugbot concludes `success`

### `CI After Gate`

An expensive workflow that:

- runs only after `Bugbot Gate` completes successfully
- contains lint, typecheck, build, Playwright, and similar costly checks

### Why this architecture

This should use `workflow_run` for the expensive workflow rather than only `needs:` inside one large workflow.

Reason:

- `needs:` orders jobs but does not prevent the parent workflow from starting
- a separate `workflow_run` hierarchy is the cleaner way to avoid wasting minutes before the gate passes

## Expected repository roles

### Shared org automation repo

This repo should contain the reusable gate implementation.

Recommended repo name:

- `.github`

Alternative:

- `platform-github`

Expected shared file:

- `.github/workflows/bugbot-gate.yml`

### Each active application repo

Each active repo should contain:

- a thin caller workflow for the shared gate
- a separate expensive CI workflow triggered only after the gate passes

Expected files in each active repo:

- `.github/workflows/bugbot-gate.yml`
- `.github/workflows/ci-after-gate.yml`

## Scope handling instructions

If the current workspace only contains the shared org repo:

- implement the shared reusable workflow there
- add a Markdown handoff file that contains the exact per-repo caller workflow, the exact downstream CI pattern, and the rollout instructions
- do **not** block on not having all downstream repos locally

If the current workspace contains one application repo instead:

- do not pretend you can implement the org repo if it is not present
- provide the per-repo caller workflow and the `workflow_run` CI split as concrete files or documentation
- clearly state that the shared reusable workflow belongs in the org automation repo

If the current workspace contains both:

- implement both layers consistently

## v1 semantics

v1 must do only this:

- accept a PR head SHA
- wait for the named Bugbot check
- pass if the check concludes `success`
- fail otherwise

v1 must **not** do this:

- inspect Bugbot comments
- inspect unresolved review threads
- parse severities
- ignore stale findings
- support manual acknowledgment exceptions

Those are future enhancements and are deliberately out of scope for now.

## Required stable contract

Treat `Bugbot Gate` as a policy contract, not an implementation detail.

That means:

- keep the gate name stable
- keep the meaning stable
- allow the implementation to change later

Today the gate is implemented by a reusable workflow.
Later the same contract may be implemented by a GitHub App.

Downstream CI should only depend on the gate result, not on how the gate was computed.

## Canonical names

Use these exact names unless the user explicitly asks for different ones:

- workflow name: `Bugbot Gate`
- job name: `Bugbot Gate`
- downstream workflow name: `CI After Gate`
- default Bugbot check name: `Cursor Bugbot`

Do not casually rename these after rollout begins, because required checks and rulesets depend on exact names.

## Reusable workflow contract

Create a reusable workflow in the shared org repo with:

- `workflow_call`
- input `sha`
- input `bugbot_check_name`
- input `timeout_minutes`

Recommended defaults:

- `bugbot_check_name: Cursor Bugbot`
- `timeout_minutes: 15`

### Example reusable workflow

```yaml
name: Bugbot Gate

on:
  workflow_call:
    inputs:
      sha:
        required: true
        type: string
      bugbot_check_name:
        required: false
        type: string
        default: Cursor Bugbot
      timeout_minutes:
        required: false
        type: number
        default: 15

permissions:
  contents: read
  checks: read
  pull-requests: read

jobs:
  gate:
    name: Bugbot Gate
    runs-on: ubuntu-latest
    timeout-minutes: ${{ inputs.timeout_minutes + 2 }}

    steps:
      - name: Wait for Bugbot check
        uses: actions/github-script@v7
        with:
          script: |
            const owner = context.repo.owner;
            const repo = context.repo.repo;
            const sha = "${{ inputs.sha }}";
            const target = "${{ inputs.bugbot_check_name }}";
            const timeoutMs = Number("${{ inputs.timeout_minutes }}") * 60 * 1000;
            const start = Date.now();

            while (true) {
              const { data } = await github.rest.checks.listForRef({
                owner,
                repo,
                ref: sha,
                per_page: 100
              });

              const matching = data.check_runs.filter(run => run.name === target);
              const latest = matching.sort((a, b) => b.id - a.id)[0];

              if (latest) {
                core.info(
                  `Latest ${target} status is ${latest.status} (conclusion: ${latest.conclusion ?? "n/a"})`
                );
              }

              if (latest && latest.status === "completed") {
                core.info(`Found completed ${target} with conclusion: ${latest.conclusion}`);

                if (latest.conclusion !== "success") {
                  core.setFailed(`${target} conclusion was ${latest.conclusion}`);
                }

                return;
              }

              if (Date.now() - start > timeoutMs) {
                core.setFailed(`Timed out waiting for ${target}`);
                return;
              }

              core.info(`Still waiting for ${target} on ${sha}...`);
              await new Promise(resolve => setTimeout(resolve, 15000));
            }
```

## Thin per-repo caller workflow

Each active repo should contain a minimal caller workflow that delegates to the shared org workflow.

Example:

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

Do not inline the full gate logic into every repo unless the user explicitly rejects reusable workflows.

## Downstream expensive CI workflow

Each active repo should move expensive checks into a second workflow triggered by `workflow_run`.

Example:

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

- if expensive CI remains on direct `pull_request`, the gate does not meaningfully save minutes
- the user’s explicit goal is to avoid running expensive checks before Bugbot is clean
- use `github.event.workflow_run.head_sha` when checking out in downstream CI so tests run on the same commit that passed the gate

## Org ruleset instructions

The organization should add a ruleset that requires the canonical gate check.

At minimum, require:

- `Bugbot Gate`

Depending on merge policy, the org may also require downstream checks such as:

- `Quality`
- `Build`
- `Playwright`

Before locking required checks into the ruleset:

1. run a real test PR
2. inspect the exact emitted check names in GitHub
3. use those exact names in the ruleset

Do not guess check names and bake them into policy without verification.

## Rollout instructions

Roll out only to active repositories first.

Recommended sequence:

1. create or update the shared org automation repo
2. add the reusable gate workflow there
3. onboard one pilot application repo
4. confirm exact check names
5. confirm expensive CI does not start before gate success
6. onboard 3-5 active repos
7. expand later only if useful

Do not force rollout to all repositories immediately. That creates admin churn on inactive repos without meaningful benefit.

## Fork PR expectation (v1)

Fork PR support in v1 should be treated as an explicit policy choice, not an assumption.

If forks are supported in v1:

- keep downstream `CI After Gate` permissions minimal and avoid secrets-dependent steps
- treat checked out fork code as untrusted when using `github.event.workflow_run.head_sha`
- verify that `Cursor Bugbot` emits the named check for fork PRs in your organization, otherwise the gate fails by design

## If only shared pieces can be implemented in the current workspace

Do **not** respond with:

> “I can only directly implement the shared pieces in this workspace. Which scope do you want me to plan for now?”

Instead:

- implement the shared reusable workflow now
- add a handoff document containing:
  - the thin per-repo caller workflow
  - the `CI After Gate` workflow pattern
  - the org ruleset steps
  - the rollout checklist

If edits are not requested, provide the exact files and instructions directly.

## Common mistakes to avoid

- leaving expensive CI on `pull_request` and expecting cost savings
- using one giant workflow with `needs:` and calling that a hierarchy
- renaming gate checks after required-check rulesets exist
- duplicating Bugbot logic independently in many repos
- starting with unresolved-comment semantics in v1
- blocking on not having every target repo available locally

## Future GitHub App compatibility

Assume a GitHub App may replace the gate later.

Design now so that migration is easy:

- keep the gate name stable
- keep the gate meaning stable
- keep logic centralized
- do not make downstream CI depend on workflow internals

Then later:

- the GitHub App can publish the same gate contract
- downstream CI and rulesets can remain largely unchanged

## Final expected outcome

The target solution should be:

1. one shared reusable Bugbot gate in the org automation repo
2. one thin caller workflow in each active repo
3. one separate expensive CI workflow triggered only after the gate passes
4. one org ruleset requiring the canonical gate check
5. a rollout that starts only with actively developed repos

If you implement or document the solution correctly, another repository-local agent should not need to ask which scope to plan for unless the user has withheld a genuinely necessary detail like the organization name or the target default branch.
