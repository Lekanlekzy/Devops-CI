# Devops-CI

Reusable GitHub Actions CI workflows for npm packages, aligned with [SLSA](https://slsa.dev) supply-chain integrity requirements.

Two `workflow_call` workflows any npm repo can adopt:
- **`npm-ci.yml`** — typecheck, a multi-version Node test matrix, build/package validation, SBOM generation, OSSF Scorecard, dependency review, and signed build provenance.
- **`security.yml`** — the compulsory merge gate: SonarCloud + Snyk, meant to be wired up as a required status check on `main`.

See [`docs/CI-USAGE.md`](docs/CI-USAGE.md) for how to call it and what it checks. CD (publishing) is not covered yet.
