# actions

Shared GitHub Actions and reusable workflows, used across personal repositories
and the `ArticulateLLM`, `InfraSonic-App`, `greekfi`, `g2i-ai` and `Div-or-Die`
organizations.

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
- uses: rockindahizzy/actions/aws-auth@v1
  with:
    role-arn: ${{ vars.AWS_ROLE_ARN }}
    region: us-west-2
```

## Pinning

Pin to a tag, or to a commit SHA where the calling workflow has elevated
permissions:

```yaml
uses: rockindahizzy/actions/aws-auth@v1        # moving tag, convenient
uses: rockindahizzy/actions/aws-auth@a1b2c3d   # immutable, safer
```

A moving `v1` means a change here reaches every consumer at once. That is the
point, and also the risk — anything running with OIDC access to AWS should pin
by SHA.

## Layout

```
<action-name>/action.yml          composite actions
.github/workflows/<name>.yml      reusable workflows
```

Composite actions wrap a sequence of steps. Reusable workflows wrap a whole job,
and are the right shape when the caller only supplies inputs.
