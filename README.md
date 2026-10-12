# enveronyx-ci-pack-shim

Public, caller-facing reusable GitHub Actions workflows for the EnverOnyx CI pack — callable from any
GitHub organization.

## Overview

This repository is the **Ci_Pack_Shim**: the public half of a two-part split. Each workflow here is a
thin caller surface that holds no proprietary logic and no secret value. At run time it:

1. authenticates to the artifact registry with the **Artifact_Registry_Credential you supply**,
2. verifies your **Licence_Entitlement** for the licensed artifact major version you pinned,
3. installs that pinned **Licensed_Artifact** (`@enveronyx/ci-pack-logic`), and
4. delegates the work to it.

The proprietary implementation lives in the licensed artifact, not here. Publishing this repository
publicly discloses nothing EnverOnyx intends to keep private.

Where it sits in the platform: `enveronyx-ci-pack` (private, the pack's own pipeline and the source of
these caller contracts) → `enveronyx-ci-pack-shim` (this repo, public) → `@enveronyx/ci-pack-logic`
(private, licence-gated) → your repository's workflows.

## Quick start

Reference a workflow by path and tag, pin the licensed artifact major version, and pass your registry
credential:

```yaml
jobs:
  build:
    uses: EnverOnyx/enveronyx-ci-pack-shim/.github/workflows/ts-build.yml@v1
    with:
      logic-version: '1'
      node-version: '22'
    secrets:
      ARTIFACT_REGISTRY_TOKEN: ${{ secrets.ENVERONYX_REGISTRY_TOKEN }}
```

Every workflow accepts the same inputs and secrets as its `enveronyx-ci-pack` counterpart, so an
existing `uses:` reference keeps resolving once the repository and tag are updated. Two inputs and one
secret are added on top of that contract:

| Name                      | Kind   | Required | Purpose                                                                       |
| ------------------------- | ------ | -------- | ----------------------------------------------------------------------------- |
| `logic-version`           | input  | yes      | Licensed artifact **major** version to install and delegate to                |
| `logic-registry-url`      | input  | no       | Registry to retrieve the artifact from (default `https://npm.pkg.github.com`) |
| `ARTIFACT_REGISTRY_TOKEN` | secret | yes      | Your client-scoped credential for that registry                               |

`ARTIFACT_REGISTRY_TOKEN` is read from your own repository or organization secrets. This repository
stores no credential of its own; the token is written to a run-scoped `.npmrc` under `RUNNER_TEMP` and
never committed.

## Available workflows

| Workflow                  | Purpose                                                                       |
| ------------------------- | ----------------------------------------------------------------------------- |
| `app-pipeline.yml`        | Docker build, ECR push, ECS deploy                                            |
| `cdk-deploy.yml`          | CDK diff, deploy, prod tag                                                    |
| `cdk-synth.yml`           | CDK synth and cloud-assembly packaging                                        |
| `ci-cdk.yml`              | Alias of `cdk-synth.yml` — see "Aliases" below                                |
| `compatibility-test.yml`  | Validate a construct-library upgrade against a consumer repo                  |
| `consumer-smoke.yml`      | Install the published packages and resolve referenced workflows as a consumer |
| `db-migration.yml`        | Flyway validate and migrate                                                   |
| `infra-pipeline.yml`      | Alias of `cdk-deploy.yml` — see "Aliases" below                               |
| `quality-gate.yml`        | ESLint, SAST, dependency scan, IaC security, SBOM                             |
| `release-integrity.yml`   | Fail the build on unresolvable ranges and dangling workflow references        |
| `repo-standards-gate.yml` | Enforce the mandatory per-repo artifacts                                      |
| `supply-chain-gate.yml`   | Vulnerability gate, allowlist, licence and typosquatting checks               |
| `supply-chain-sbom.yml`   | SBOM generation and SLSA provenance                                           |
| `supply-chain-sign.yml`   | Lambda and container image signing, signature verification                    |
| `ts-build.yml`            | TypeScript compile and asset bundling                                         |
| `ts-publish.yml`          | Publish to the package registry and tag                                       |
| `ts-test.yml`             | Test suite with optional sharding and coverage                                |

### Aliases

`infra-pipeline.yml` and `ci-cdk.yml` are **published aliases**. Both names were referenced by platform
repositories and documentation while no workflow of either name existed at any ref, so every one of those
references failed at workflow resolution. Each alias declares the caller contract of the workflow it
aliases and delegates to the same licensed-artifact step body, so an alias cannot drift from its target.
Prefer `cdk-deploy.yml` and `cdk-synth.yml` in new repositories.

One caveat on `ci-cdk.yml`: four of its documented inputs — `skip-build`, `skip-tests`, `test-command`,
and `publish` — have no counterpart in `cdk-synth.yml`. They are accepted so an existing caller does not
fail outright, but they are not honoured silently. The first three warn in the run log; `publish: true`
**fails the run**, because completing without the publication you asked for is the worse outcome. Compose
`ts-build.yml`, `ts-test.yml`, and `ts-publish.yml` for those stages.

The authoritative statement of which pack workflows are republished here is
[`docs/shim-split-inventory.md`](https://github.com/enveronyx/enveronyx-ci-pack/blob/main/docs/shim-split-inventory.md)
in `enveronyx-ci-pack`. The pack's own pipeline (`ci.yml`) and its operator-only DR failover workflow
are deliberately not republished.

## Build / test

There is nothing to compile — this repository contains workflow YAML only. It is validated by
GitHub's own workflow parser when a caller invokes a workflow, and by the shim contract tests in
`enveronyx-ci-pack`.

## Key conventions

- **No logic.** A shim workflow retains exactly four concerns: registry authentication, the licence
  entitlement check, the pinned artifact install, and the delegation call. Anything else belongs in
  the licensed artifact.
- **No secret value.** Only `${{ secrets.* }}` references, never a literal credential. This repository
  is public.
- **Entitlement before install.** The entitlement check runs ahead of the install so a licence problem
  is reported as a licence problem rather than an opaque install error.
- **Actions pinned to a 40-character SHA** with a trailing version comment, matching the pack. This
  repository is a supply-chain root of trust for every caller.
- **Caller contract is the pack's.** Adding, renaming, or retyping an input is a breaking change for
  callers in client-owned organizations. Preserve the contract.

## Versioning

Callers reference a tag (`@v1`), not a branch. The shim major version maps to supported licensed
artifact majors through the platform compatibility matrix.

**No tag is published in this repository yet**, so the `@v1` examples above do not resolve until one is
cut. Cut it as a **lightweight** tag rather than an annotated one: the two aliases in `enveronyx-ci-pack`
delegate with a same-repository reference, and GitHub has not resolved that form through an annotated tag.

## Licence

The workflows in this repository are published for use by EnverOnyx clients. The logic they delegate
to is licensed separately and requires a valid entitlement.
