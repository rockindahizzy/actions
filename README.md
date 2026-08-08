# actions

Shared GitHub Actions and reusable workflows, used across personal repositories
and five organizations.

**This repository is public deliberately.** GitHub does not allow private
repositories to share actions across owners — a private repo here could not be
consumed by the organizations that use it. Public is what makes the sharing
work without putting a token in every consuming repository.

## Nothing sensitive lives here

Actions contain logic only. Account IDs, role ARNs, bucket names, hostnames and
anything else naming private infrastructure are **inputs supplied by the
caller**, never values committed here.

Before merging anything, check it does not hardcode:

- AWS account IDs or role ARNs
- S3 bucket names or other resource identifiers
- Internal hostnames or endpoints
- Any secret, token or credential

## Usage

```yaml
- uses: rockindahizzy/actions/aws-auth@aws-auth/v0
  with:
    role-arn: ${{ vars.AWS_ROLE_ARN }}
    region: us-west-2
```

## Versioning

Each action versions independently, and its tags carry its name:

```
aws-auth/v0.2.1          exact version
aws-auth/v0              moving, tracks the latest 0.x
setup-node-yarn/v0.1.0   unrelated to aws-auth's version
```

Versions are computed from conventional commit messages, never chosen by hand. A
breaking change to one action does not bump any other.

Actions start at `v0` and stay there while their interface settles. Under `0.x` a
breaking change bumps the minor, so **`aws-auth/v0` will carry breaking changes
to anyone tracking it**. That is what `v0` means here — pin exactly, or pin by
SHA, if that is not acceptable.

There is no LTS. When an action reaches a new major, the previous one receives
nothing further, including security fixes.

See [ADR 0001](docs/adr/0001-per-action-semantic-versioning.md) for the reasoning.

## Pinning

Pin to a tag, or to a commit SHA where the calling workflow has elevated
permissions:

```yaml
uses: rockindahizzy/actions/aws-auth@aws-auth/v0   # moving tag, convenient
uses: rockindahizzy/actions/aws-auth@a1b2c3d       # immutable, safer
```

A moving tag means a change here reaches every consumer at once. That is the
point, and also the risk — anything running with OIDC access to AWS should pin
by SHA.

## Layout

```
aws-auth/
  action.yml                      the action itself
  README.md

terraform-plan/
  README.md                       no action.yml — see below
.github/workflows/
  terraform-plan.yml              the workflow consumers call
```

Composite actions wrap a sequence of steps and drop into an existing job.
Reusable workflows wrap a whole job, bringing their own `runs-on`,
`permissions` and `concurrency` — the right shape when a job needs privileges
of its own, or must be split across several jobs.

### Why reusable workflows have a directory with no `action.yml`

Two separate constraints collide:

- GitHub requires reusable workflows to live flat in `.github/workflows/`.
  [Subdirectories are not supported](https://github.com/actions/runner/issues/2102),
  so `terraform-plan/workflow.yml` would simply never be found.
- release-please can only version **directories**, never single files.

Per-workflow versions therefore need a directory that exists purely as the
release component. It holds a README and, once released, the generated
`CHANGELOG.md`. The workflow itself lives in `.github/workflows/`.

This is a deliberate trade recorded in
[ADR 0001](docs/adr/0001-per-action-semantic-versioning.md): every artifact
versions the same way, bought with a directory that will look pointless to
anyone who does not know why. CI fails if a component and its callable file
drift apart, since nothing else would notice.

A composite action could not do this job anyway — it cannot declare
`permissions:` or `concurrency:`, and `terraform-plan` depends on both. Its
fork-PR refusal works because the authorising job holds no `id-token`, which
is a job-level boundary a composite action has no way to express.

### Referencing each kind

```yaml
# composite action — a step inside your job
- uses: rockindahizzy/actions/aws-auth@aws-auth/v0

# reusable workflow — a whole job
jobs:
  plan:
    uses: rockindahizzy/actions/.github/workflows/terraform-plan.yml@terraform-plan/v0
```

The workflow ref names the workflow twice, once in the path and once in the
tag. That is awkward but unavoidable: the path locates the file and the tag
selects the version, and per-workflow versioning needs both.
