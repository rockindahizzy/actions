# aws-auth

Assume an AWS IAM role from GitHub Actions using OIDC, so no long-lived access
keys live in repository secrets.

A thin wrapper over
[`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials),
SHA-pinned here so every consumer moves together. The value is centralising the
version and the calling convention, not the logic — plus a preflight check that
turns the most common misconfiguration into a message that names its own cause.

## Usage

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # required — see below
      contents: read
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      - uses: rockindahizzy/actions/aws-auth@aws-auth/v0
        with:
          role-arn: ${{ vars.AWS_ROLE_ARN }}
          region: us-west-2

      # Subsequent steps are authenticated.
      - run: aws s3 sync ./dist "s3://${{ vars.ARTIFACT_BUCKET }}/"
```

## `permissions: id-token: write` is required

Without it GitHub issues no OIDC token, there is nothing to exchange for AWS
credentials, and the failure that surfaces from AWS does not mention the
permission.

This action checks for the token up front and fails with an explicit message
instead. Three things are worth knowing:

- Setting `permissions:` at all is opt-in-only. A job declaring
  `permissions: contents: read` and nothing else **loses** `id-token`. List both.
- Setting it on the job is preferable to the workflow top level — it keeps the
  elevated permission scoped to the job that needs it.
- `pull_request` runs triggered **from a fork** never receive an OIDC token.
  Use `pull_request_target`, or restrict deployment to post-merge, if you need
  credentials for contributions from forks.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `role-arn` | yes | — | ARN of the role to assume. Supply from `vars`/`secrets`; never hardcode. |
| `region` | yes | — | Region to configure, e.g. `us-west-2`. Sets `AWS_REGION` and `AWS_DEFAULT_REGION`. |
| `role-session-name` | no | `gha-<repo-id>-<run-id>` | Session name, as it appears in CloudTrail. Truncated to STS's 64-character limit. |
| `audience` | no | `sts.amazonaws.com` | OIDC `aud` claim. Change only if the trust policy expects something else. |
| `mask-aws-account-id` | no | `false` | Mask the account ID and ARN in logs. Must be exactly `true` or `false`. |
| `role-duration-seconds` | no | `3600` | Session lifetime, 900–43200. Must not exceed the role's `MaxSessionDuration`. |

## Outputs

| Output | Description |
|---|---|
| `account-id` | Account ID the credentials belong to. Empty when `mask-aws-account-id` is `true` — a masked value cannot cross a step output. |
| `preflight-passed` | `true` when validation and the OIDC check passed. Exists for this repo's self-test; not intended for general use. |

## The AWS side

This action assumes the AWS account already trusts this repository. That setup
is Terraform work and lives in the infrastructure repository, not here:

1. A GitHub OIDC identity provider for `token.actions.githubusercontent.com`.
2. A role whose trust policy admits the specific repositories that may assume it.

**Scope the `sub` claim.** A wildcard trust means any repository in the org — or
on GitHub — can assume the role:

```jsonc
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  },
  "StringLike": {
    // Scope to the repo. Narrow to :ref:refs/heads/main where the workflow
    // only ever runs post-merge.
    "token.actions.githubusercontent.com:sub": "repo:YourOrg/your-repo:*"
  }
}
```

Grant the role only the permissions the workflow actually uses.

## Pinning

`aws-auth/v0` moves, and under `0.x` a breaking change bumps the minor — so the
moving tag carries breaking changes. Any workflow holding OIDC access to AWS
should pin by SHA:

```yaml
uses: rockindahizzy/actions/aws-auth@aws-auth/v0   # moving, convenient
uses: rockindahizzy/actions/aws-auth@<full-sha>    # immutable, safer
```

See [ADR 0001](../docs/adr/0001-per-action-semantic-versioning.md).
