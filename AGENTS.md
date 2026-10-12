# AGENTS.md — enveronyx-ci-pack-shim

## Purpose

The **public** caller-facing half of the EnverOnyx CI pack. It holds 17 reusable `workflow_call`
workflows that any GitHub organization can invoke. Each one authenticates to the artifact registry with
the caller's `ARTIFACT_REGISTRY_TOKEN`, verifies the caller's licence entitlement, installs the pinned
`@enveronyx/ci-pack-logic` licensed artifact, and delegates the work to it. The proprietary
implementation is not in this repository and must never be added to it.

In the delivery chain: `enveronyx-ci-pack` (private; owns the caller contracts and the pack's own
pipeline) → **this repo** (public caller surface) → `@enveronyx/ci-pack-logic` (private, licence-gated)
→ the client's own repository workflows.

## Architecture

```
.github/workflows/      17 shim workflows, one per caller-facing pack workflow
                        (includes the ci-cdk / infra-pipeline aliases)
.github/CODEOWNERS      review routing
catalog-info.yaml       Backstage component descriptor
README.md               caller-facing usage
AGENTS.md               this file
```

There is no source tree, no build, and no package manifest. Every file is declarative.

Each workflow has exactly one job (`delegate`) with exactly six steps, in this order:

1. `Checkout caller repository` — `actions/checkout`
2. `Set up Node.js` — `actions/setup-node`
3. `Authenticate to the Licensed_Artifact registry` — writes a run-scoped `.npmrc` under `RUNNER_TEMP`
   from `secrets.ARTIFACT_REGISTRY_TOKEN` and exports `NPM_CONFIG_USERCONFIG`
4. `Verify Licence_Entitlement` — `npm view` against the pinned major; runs **before** the install
5. `Install the pinned Licensed_Artifact` — `npm install --no-save @enveronyx/ci-pack-logic@^<major>`
6. `Delegate to ci-pack-logic <entry-point>` — `npx --no-install ci-pack-logic <entry-point>`

Caller inputs reach the artifact as `CI_PACK_INPUT_<UPPER_SNAKE>` environment variables, and caller
secrets as `CI_PACK_SECRET_<NAME>`. Workflow-level outputs are surfaced from the delegation step's
outputs through the `delegate` job.

## Dependencies

### Internal (platform)

- `@enveronyx/ci-pack-logic` — the licensed artifact holding all extracted logic. Installed at run
  time, pinned by the caller's `logic-version` input. Not yet published; see Cold Gaps.
- `enveronyx-ci-pack` — the private pack. It owns the caller contracts these workflows mirror and the
  `docs/shim-split-inventory.md` declaration of which files are republished here.

### External (third-party)

- `actions/checkout` — checks out the **caller's** repository (a reusable workflow runs in the caller's
  context, so this is the caller's code, not this repo's)
- `actions/setup-node` — provides the Node.js runtime the artifact install and delegation run under

### Consumed by

- Client infrastructure repositories in client-owned GitHub organizations
- `enveronyx-client-infra-live-template` (generated workflows reference the shim at a real tag)
- EnverOnyx platform repositories, once migrated off direct pack references

## External Actors & Integrations

| System                                     | Direction | Protocol        | Auth                                                                                                               |
| ------------------------------------------ | --------- | --------------- | ------------------------------------------------------------------------------------------------------------------ |
| Package registry (default GitHub Packages) | out       | HTTPS / npm     | Caller-supplied `ARTIFACT_REGISTRY_TOKEN` in a run-scoped `.npmrc`                                                 |
| GitHub Actions runner                      | in        | `workflow_call` | Caller's workflow token and GitHub Environments                                                                    |
| AWS                                        | out       | AWS APIs        | OIDC role assumption via the caller's `OIDC_ROLE_ARN` secret, performed **inside** the licensed artifact, not here |

This repository performs no AWS call of its own. Workflows whose pack counterpart assumed a role still
accept the same `OIDC_ROLE_ARN` secret and forward it; the assumption happens in the artifact.

## Conventions & Gotchas

- **This repository is public.** Never add a literal credential, an internal hostname, an account
  identifier, a client name, or proprietary logic. Only `${{ secrets.* }}` references.
- **Six steps, no more.** A shim retains registry auth, the entitlement check, the pinned install, and
  the delegation call. Adding a build, test, deploy, or scan step here puts proprietary logic in a
  public repository.
- **The caller contract is inherited, not designed here.** Inputs, secrets, outputs, types, defaults,
  and `required` flags mirror `enveronyx-ci-pack`. Changing one breaks callers in organizations the
  Platform does not control. Change the pack first.
- **Entitlement check precedes the install.** Reordering them makes a licence failure surface as an
  opaque install error.
- **Job id is `delegate`, deliberately.** `jobs.<id>` inside a `${{ }}` expression must not contain a
  hyphen, or `jobs.cdk-synth` parses as a subtraction. Do not rename it to the workflow name.
- **Actions pinned to a 40-character SHA** with a trailing `# vX.Y.Z` comment. This repository is a
  supply-chain root of trust for every client caller; a tag reference is not acceptable here.
- **No relative `uses:`.** A reusable workflow runs in the caller's checkout, so `./.github/...` would
  resolve against the caller's repository, not this one.
- **`consumer-smoke.yml` keeps only its `workflow_call` surface here.** Its `schedule` and
  `workflow_dispatch` triggers stay in the pack, which runs its own smoke check.

## Build, Test, Deploy

No build step and no test runner. Green means:

- every workflow file parses as valid YAML and declares an `on: workflow_call:` trigger;
- every pack input, secret, and output still appears with the same type, `required`, and default;
- `logic-version` is present and required, and `ARTIFACT_REGISTRY_TOKEN` is present and required;
- every `uses:` is SHA-pinned with a version comment;
- no file contains a literal secret value.

These are asserted by the shim contract tests in `enveronyx-ci-pack` (task 14.4). Deployment is a git
tag: callers are intended to reference `@v1`, so a change reaches them only when the tag moves or a new
major is cut. **No tag exists in this repository yet** — cutting the first `v1` is a remote action the
Platform owner performs (see `PLACEHOLDER_TAG_V1` under Cold Gaps), and until then the README's `@v1`
examples do not resolve.

## Cold Gaps

- `PLACEHOLDER_PACKAGE_CI_PACK_LOGIC` — `@enveronyx/ci-pack-logic` is not published yet. The step
  bodies **have** been extracted (task 14.3 is done): they live in `enveronyx-ci-pack` under
  `packages/ci-pack-logic`, with one entry point per shim workflow. What remains is the publish to the
  private registry — a remote action the Platform owner performs. Until that publish lands, the
  install and delegation steps cannot succeed against a real registry.
- `PLACEHOLDER_ENTITLEMENT_MECHANISM` — the entitlement check currently infers entitlement from
  registry read access. The explicit named-entitlement failure message and its stated remediation
  (requirement 17.6) are not implemented yet.
- `PLACEHOLDER_TAG_V1` — no `v1` tag exists in **this** repository yet, so the README's `@v1` examples
  do not resolve until one is cut. Cutting it is a remote action the Platform owner performs; task 14.5
  did the in-repo half. Cut it as a **lightweight** tag, not an annotated one: `ci-cdk.yml` and
  `infra-pipeline.yml` in `enveronyx-ci-pack` delegate with a same-repository reference, and GitHub has
  not resolved that form through an annotated tag.
  The `infra-pipeline` / `ci-cdk` aliases themselves have landed — they exist here as shim workflows and
  in `enveronyx-ci-pack` as published aliases of `cdk-deploy.yml` and `cdk-synth.yml` (task 14.5,
  R12.4/12.5). Nothing in `enveronyx-ci-pack` references a `v1` tag any more.
- `PLACEHOLDER_REGISTRY_ENTITLEMENT_SCOPE` — whether a per-client registry scope or a separate
  entitlement service gates access has not been decided.
