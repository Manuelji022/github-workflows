# GitHub Workflows

Centralized, reusable GitHub Actions workflows for consistent CI and pull-request standards across my projects.

This repository owns implementation. Consumers own triggers, dependencies, scripts, project configuration, secrets, and deployment configuration. All workflows use `workflow_call` only, run at the repository root on GitHub-hosted `ubuntu-24.04`, and require no supplied secrets.

## Available workflows

### Node CI

`node-ci.yml` runs independent `lint`, `typecheck`, `test`, and `build` jobs. Every enabled job installs locked dependencies and fails on missing scripts or command failures. Build validates compilation only. npm uses `npm ci`; pnpm uses `pnpm install --frozen-lockfile`.

### Python CI

`python-ci.yml` runs independent Ruff, ty, and pytest jobs. uv 0.12.12 installs the requested managed Python, then runs `uv sync --locked --group dev`. Checks use `uv run --no-sync ruff check .`, `uv run --no-sync ty check`, and `uv run --no-sync pytest` so execution cannot update the lockfile or resynchronize the environment. Each enabled tool must exist in `.venv/bin`.

### PR Policy

`pr-policy.yml` validates PR metadata without checking out code. It produces a job summary and actionable error/warning annotations, without posting comments or changing labels.

Titles use `type: description` or `type(scope): description`, optionally with `!` before the colon. Fixed V1 types: `feat`, `fix`, `refactor`, `test`, `docs`, `build`, `ci`, `chore`, `perf`.

The body requires non-empty `Summary`, `Why`, `Scope`, `Out of scope`, `How to test`, and `Risk` sections. Use Markdown ATX headings (`#` through `######`); matching ignores case, repeated whitespace, and optional closing hashes. HTML comments and fenced heading examples do not satisfy section requirements. Sections end at the next heading; put explanatory text directly below each required heading. Placeholder-only content such as TODO, TBD, N/A, or comments fails. `None`, `N/A`, and `Not applicable` are accepted for Out of scope and Risk. Setext headings are outside V1.

Review size uses additions **plus** deletions from GitHub's paginated PR files API. “Source” conservatively means every changed file, including documentation/configuration/binaries, except these exact basenames at any depth: `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `uv.lock`. Excluded totals are reported separately. Renames are excluded only when both old and new names are lockfiles. No directories or other generated artifacts are excluded. Binary files count as files even when GitHub reports zero LOC.

| Size | Result |
| --- | --- |
| 0–400 LOC and ≤10 source files | NORMAL; allowed |
| 401–800 LOC or >10 source files | LARGE; visible warning, allowed |
| >800 LOC | OVERSIZED; requires a non-placeholder `## Large PR justification` section |

The policy checks presence, not explanation quality or semantic correctness. It fails if metadata changes during evaluation or the file list is incomplete, including PRs over GitHub's 3,000-file API limit. Rerun transient failures; split PRs beyond that limit. It reads current title/body so reruns can validate edits, and verifies the event's head commit still matches.

## Usage

**The `v1` tag does not exist yet.** Follow the [release procedure](RELEASE.md) to publish the first compatibility tag after the implementation is reviewed, or substitute a reviewed immutable commit SHA until then. Enable access to this reusable repository in GitHub Actions settings if it is private.

### Node caller

```yaml
name: Node CI
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
jobs:
  node-ci:
    uses: Manuelji022/github-workflows/.github/workflows/node-ci.yml@v1
    with:
      node-version: "22"
      package-manager: pnpm
```

### Python caller

```yaml
name: Python CI
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
jobs:
  python-ci:
    uses: Manuelji022/github-workflows/.github/workflows/python-ci.yml@v1
    with:
      python-version: "3.13"
```

### PR policy caller

```yaml
name: PR Policy
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]
permissions:
  pull-requests: read
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
jobs:
  policy:
    uses: Manuelji022/github-workflows/.github/workflows/pr-policy.yml@v1
```

`edited` covers title/body edits, `synchronize` new commits, and `opened`/`reopened` the PR lifecycle. Draft PRs are checked too, so `ready_for_review` is unnecessary. See [GitHub PR events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request).

### Concurrency and merge gates

Concurrency belongs **only in callers**. Give each caller workflow a distinct name. The group above isolates PR numbers and branch refs, cancels obsolete runs of the same workflow, and lets different PRs run independently. If combining CI and policy in one caller, enable `edited` there too.

The called workflow inherits the caller's `github` context: `github.event` is the caller event, `github.event.pull_request` is its PR payload, `github.workflow` is the caller's name, `github.head_ref` is the PR source branch, and `github.ref` is normally `refs/pull/<number>/merge` for these PR events (or the pushed branch ref). GitHub warns that identical caller/callee concurrency groups can cancel the caller itself. See [reusable workflow semantics](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations) and [context definitions](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts#github-context).

Require the resulting enabled CI checks and PR policy check in branch protection or a repository ruleset: PR → CI and policy → required status checks → merge. Select the actual check names after the first run. Disabled jobs are skipped and are not evidence of validation. Repository settings are not configured here. V1 does not implement merge-queue integration.

## Inputs

All inputs are optional. Runtime versions are strings; run switches are booleans; thresholds are non-negative integer-valued numbers.

| Workflow | Input | Default |
| --- | --- | --- |
| Node | `node-version` | `"22"` |
| Node | `package-manager` | `npm` (only npm/pnpm supported) |
| Node | `run-lint`, `run-typecheck`, `run-tests`, `run-build` | `true` each |
| Python | `python-version` | `"3.13"` |
| Python | `run-lint`, `run-typecheck`, `run-tests` | `true` each |
| Policy | `large-loc-threshold` | `400` |
| Policy | `oversized-loc-threshold` | `800` |
| Policy | `large-files-threshold` | `10` |

Oversized LOC must exceed large LOC. Comparisons are strictly greater-than. Disable checks explicitly with e.g. `run-build: false`. With all checks disabled, no CI jobs run (including input validation).

## Required project configuration

### Node

Commit root `package.json` and `package-lock.json` for npm, or `pnpm-lock.yaml` for pnpm. Declare real commands for every enabled script:

```json
{
  "scripts": {
    "lint": "...",
    "typecheck": "...",
    "test": "...",
    "build": "..."
  }
}
```

Replace the ellipses with project commands; tests must terminate rather than watch. pnpm additionally requires `"packageManager": "pnpm@X.Y.Z"` with the project's exact stable version; an integrity hash is supported and recommended. No global pnpm version is selected by this repository.

[Corepack](https://github.com/nodejs/corepack#readme) is bundled with official Node 22/24 distributions but must be enabled. V1 enables it before [setup-node caching](https://github.com/actions/setup-node/tree/v7.0.0#caching-global-packages-data), which requires the package manager to be available. The second setup-node invocation restores the npm/pnpm download/store cache, never `node_modules`. pnpm overrides must select a distribution with Corepack; Node 25+ no longer bundles it and is outside V1 pnpm support. npm can use other supported Node versions. Pin a full Node patch version for tighter runtime reproducibility.

### Python

Commit root `pyproject.toml` and `uv.lock`. Declare `ruff`, `ty`, and `pytest` in `[dependency-groups].dev`, then lock them with uv. The project owns tool versions and configuration; CI never separately installs tools. The Python input overrides `.python-version` and must satisfy the project's Python constraint. Pin a full Python patch version for tighter runtime reproducibility.

Local equivalents:

```bash
uv python install 3.13
uv sync --locked --group dev
uv run --no-sync ruff check .
uv run --no-sync ty check
uv run --no-sync pytest
```

`--locked` validates that the committed lock is up to date with project metadata and fails if it would need updating. Update and commit the lock locally when changing project metadata. The explicit dev group prevents custom default groups from omitting CI tools. uv caches downloads through setup-uv; virtual environments are not shared across jobs. See [Astral's CI guide](https://docs.astral.sh/uv/guides/integration/github/), [sync semantics](https://docs.astral.sh/uv/concepts/projects/sync/), and [ty CLI](https://docs.astral.sh/ty/reference/cli/).

## Security

CI grants only `contents: read`; checkout disables credential persistence. Policy grants only `pull-requests: read`, needed for the files API. Callers must permit these scopes: reusable workflows can reduce but cannot elevate caller permissions. No workflow requires custom secrets or `secrets: inherit`; do not supply secrets to untrusted fork code or run these CI workflows from privileged events. CI intentionally executes project dependencies/scripts with read-only access.

PR metadata is parsed as JSON in a Python script, never interpolated into executable shell source. Policy does not print raw PR text/filenames into annotations or summaries and never executes project code. It uses the automatic scoped GitHub token only for API reads. See [GitHub reuse and secrets documentation](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows).

External Actions were verified against official release/tag APIs on 2026-09-10 and are pinned to the matching full commit SHA with version comments:

| Action | Release |
| --- | --- |
| GitHub checkout | [v7.0.1](https://github.com/actions/checkout/releases/tag/v7.0.1) |
| GitHub setup-node | [v7.0.0](https://github.com/actions/setup-node/releases/tag/v7.0.0) |
| Astral setup-uv | [v10.0.1](https://github.com/astral-sh/setup-uv/releases/tag/v10.0.1) |

These Actions use Node 24 internally, independently of the project's Node version; current hosted runners meet the minimum runner requirement (v2.327.1). Hosted images and runtime minor-version selectors still evolve, so lockfiles and Action pins are not a hermetic build guarantee.

## Versioning

Future `@v1` tags provide a stable compatibility line that may advance to compatible fixes. Consumers needing immutable behavior should reference a reviewed full commit SHA. No tag or release automation is created in V1.

Treat Action updates as dependency updates: verify the official stable release, resolve its commit, check runtime compatibility, and update both SHA and version comment. Standard `uses:` entries allow later automated updates.

V1 deliberately repeats setup for parallel, independent feedback. It has no framework detection, subdirectory/monorepo orchestration, formatting gate, deployment, release automation, scanning, containers, E2E testing, or dependency automation. Those belong in separate changes.
