# actions

Shared GitHub Actions and reusable workflows.

**This repository is public deliberately.** GitHub does not allow private
repositories to share actions across owners — a private repo here could not be
consumed by any of the organizations above. Public is what makes the sharing
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
<action-name>/action.yml          composite actions
.github/workflows/<name>.yml      reusable workflows
```

Composite actions wrap a sequence of steps. Reusable workflows wrap a whole job,
and are the right shape when the caller only supplies inputs.
