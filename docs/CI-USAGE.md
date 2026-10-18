# npm CI reusable workflows

This repository hosts two reusable GitHub Actions workflows for npm-based repos:

- **[`npm-ci.yml`](../.github/workflows/npm-ci.yml)** — build correctness: typecheck, a Node version matrix of tests, build/package validation, SBOM generation, and signed SLSA build provenance. Steps skip gracefully when a repo hasn't set up the relevant script/config.
- **[`security.yml`](../.github/workflows/security.yml)** — the compulsory quality/security gate: SonarCloud + Snyk. Unlike `npm-ci.yml`, this one fails loudly (missing test script, missing/invalid tokens) rather than skipping, since it's meant to be a required status check before merging to `main`.

CD (publishing) is not covered yet and will be added separately.

## Why two separate workflows

`npm-ci.yml` answers "does the code build and work" — it's useful even for a repo with no security tooling configured yet, so its jobs degrade gracefully. `security.yml` answers "is this safe/clean enough to merge" — since you want that to be non-optional, it's kept in its own file with `required: true` secrets/inputs, so a caller repo that hasn't configured SonarCloud/Snyk gets a hard failure instead of a silently-skipped check. Splitting them also means `security.yml` can be added as a required branch-protection check independently of whichever `npm-ci.yml` jobs a given repo happens to trigger. `security.yml` runs its own install + test pass to produce coverage for SonarCloud rather than depending on `npm-ci.yml`'s run — the two workflows are otherwise unrelated GitHub Actions runs and can't share job outputs, so keeping it self-contained avoids a fragile cross-workflow artifact fetch at the cost of running tests twice per PR.

## What `npm-ci.yml` does

| Job | What it checks | Runs when |
|---|---|---|
| `detect` | Package manager (npm/yarn/pnpm), which optional scripts/configs exist | Always |
| `typecheck` | `tsc --noEmit` | `tsconfig.json` present |
| `test` | Test script across a Node version matrix | `test` script present |
| `build` | Build script, `publint`, `arethetypeswrong`, packs a tarball, generates an SBOM | `run-build` input true and `build` script present |
| `bundle-size` | `size-limit` | `size-limit` is a dependency |
| `dependency-review` | Flags newly introduced vulnerable/risky dependencies in the PR diff | Pull requests only |
| `scorecard` | OSSF Scorecard supply-chain self-assessment | Push to the default branch only |
| `provenance` | Signed SLSA build provenance + SBOM attestation for the packed tarball | `build` job ran |

**Removed from the original design, and why:**
- **Lint** — dropped. It's a code-quality nicety, not something this pipeline needs to own; add it in your own repo's workflow if wanted.
- **CodeQL** — dropped. GitHub's **Default Setup** (Security tab → Code security → Code scanning) already runs CodeQL automatically with no workflow needed, and GitHub blocks running a custom CodeQL config alongside Default Setup for the same repo. Enable Default Setup per-repo instead. If you're on Advanced Setup (custom queries/paths) or a plan where Default Setup isn't available, re-add a `codeql` job.
- **Dependency audit job** (`npm audit`, `audit signatures`, `lockfile-lint`, `license-checker`) — dropped as a full job since Snyk (in `security.yml`) already covers vulnerability scanning with a larger, more actively maintained database. SBOM generation — which that job also did — moved into the `build` job, since `provenance` still needs it for the SBOM attestation.
- `dependency-review` is kept even though the audit job was dropped: it checks the *diff* of a specific PR for newly introduced risky/vulnerable dependencies (a GitHub-native, zero-config check), which is a different, narrower job than a full lockfile audit.

## What `security.yml` does

| Job | What it checks |
|---|---|
| `detect` | Package manager + fails loudly if no `test` script exists (SonarCloud needs coverage) |
| `sonarcloud` | Runs tests with coverage, then a SonarCloud quality gate on your own code — bugs, code smells, duplication, coverage gaps |
| `snyk` | Scans your dependency tree for known CVEs |

Both `SONAR_TOKEN`/`SNYK_TOKEN` secrets and `sonar-project-key`/`sonar-organization` inputs are `required: true` — omitting them fails the calling workflow at validation time (before any job runs), not silently.

## Making the security gate compulsory

Declaring secrets/inputs as `required: true` only fails a caller that forgets to pass them — it does not, by itself, block a PR from merging. To actually make `security.yml` compulsory:
1. Add a caller job for it in your repo's `.github/workflows/ci.yml` (see below).
2. In your repo's branch protection rules for `main`, add the `sonarcloud` and `snyk` job names (from that caller workflow) as **required status checks**.

## Calling these from your repo

Create `.github/workflows/ci.yml` in your npm project:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: <org>/Devops-CI/.github/workflows/npm-ci.yml@<commit-sha>
    with:
      node-versions: '["20", "18", "22"]'
      run-build: true
      package-manager: auto

  security:
    uses: <org>/Devops-CI/.github/workflows/security.yml@<commit-sha>
    with:
      node-version: "20"
      package-manager: auto
      sonar-project-key: "<org>_<repo>"
      sonar-organization: "<org>"
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

Pin `@<commit-sha>` to a specific commit in this repo (not a branch name), for the same SLSA build-integrity reasons every action inside these workflows is itself pinned to a SHA. Dependabot (configured here via `github-actions` ecosystem updates) does not follow this cross-repo reference automatically — bump it manually, or watch this repo's releases.

## Inputs

### `npm-ci.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `working-directory` | string | `.` | Directory containing `package.json`, for monorepos |
| `node-versions` | string (JSON array) | `'["20", "18", "22"]'` | Test matrix; **first entry** is also used for `typecheck`/`build`/`bundle-size` |
| `run-build` | boolean | `true` | Set `false` for packages with no build step |
| `package-manager` | string | `"auto"` | `npm`, `yarn`, `pnpm`, or `auto` to detect from the lockfile |

### `security.yml`

| Input | Type | Default | Required |
|---|---|---|---|
| `working-directory` | string | `.` | No |
| `node-version` | string | `"20"` | No |
| `package-manager` | string | `"auto"` | No |
| `sonar-project-key` | string | — | **Yes** |
| `sonar-organization` | string | — | **Yes** |

## Secrets

| Secret | Used by | Required |
|---|---|---|
| `SONAR_TOKEN` | `security.yml` (create a project at [sonarcloud.io](https://sonarcloud.io), generate a token) | **Yes** |
| `SNYK_TOKEN` | `security.yml` (create a token at [snyk.io](https://snyk.io)) | **Yes** |

## Requirements on the caller repository

- `package-lock.json`, `yarn.lock`, or `pnpm-lock.yaml` committed (used for package manager auto-detection and install caching), unless `package-manager` is set explicitly.
- If using yarn or pnpm without a `"packageManager"` field in `package.json`, both workflows activate the latest version via Corepack automatically — pinning an exact version with `"packageManager"` is recommended for reproducible builds.
- A `test` script for `test`/`sonarcloud`; a `build` script for `build`/`provenance`.
- `security.yml`'s Snyk/coverage step assumes **Yarn Berry** (not Yarn Classic) if `package-manager` resolves to `yarn` — that's what Corepack activates by default. Pin `"packageManager": "yarn@1.x"` and you'll need a different audit invocation if staying on Classic.

## SLSA alignment

- Every action referenced by these workflows (including the shared composite action) is pinned to a full commit SHA, not a mutable tag.
- Builds run on GitHub-hosted, ephemeral runners — no self-hosted/persistent infrastructure.
- `actions/attest-build-provenance` and `actions/attest-sbom` (in `npm-ci.yml`'s `provenance` job) produce signed, non-falsifiable in-toto attestations (via Sigstore/OIDC) tying the packed tarball to the exact workflow run, commit, and inputs that produced it — viewable under your repo's **Attestations** tab.
- An SBOM (CycloneDX, via `cdxgen`) is generated in the `build` job on every run, feeding both the SBOM attestation and general traceability.

This combination gets a caller repo close to **SLSA Build Level 3**: hosted, isolated builds; signed, tamper-resistant provenance; and a fully scripted (non-manual) build process. Reaching Level 3 formally also requires the caller repo's own `build`/`test` scripts to be side-effect-free and not depend on secrets beyond what's declared here.
