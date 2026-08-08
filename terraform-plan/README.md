# terraform-plan

Plans one Terraform environment on a pull request, posts the result as a PR
comment, and **fails the check if the plan destroys anything** unless the PR
carries the `allow-destroy` label.

The workflow file consumers call is
[`.github/workflows/terraform-plan.yml`](../.github/workflows/terraform-plan.yml).

## Why this directory exists

It is the versioned component for that workflow, and it contains no executable
code.

Reusable workflows must live flat in `.github/workflows/` — GitHub does not
support subdirectories there — and release-please can only version directories,
never single files. The two constraints are incompatible, so each reusable
workflow gets a directory that acts as its release component while the callable
file stays where GitHub requires it.

```
terraform-plan/                    versioned component -> terraform-plan/v0.1.0
.github/workflows/terraform-plan.yml   the file consumers call
```

Nothing inherently keeps the two in step, so `ci.yml` has a
`reusable-workflow-components` check that fails if a component directory has no
matching workflow file or the reverse. See
[ADR 0001](../docs/adr/0001-per-action-semantic-versioning.md).

## Usage

One job per environment. The job name is yours; the `name` input is what
comment commands target.

```yaml
# .github/workflows/terraform.yml in the consuming repository
name: terraform

on:
  pull_request:
  issue_comment:
    types: [created]

# No blanket grant at the top; each calling job grants exactly what it needs.
permissions: {}

jobs:
  development:
    uses: rockindahizzy/actions/.github/workflows/terraform-plan.yml@terraform-plan/v0
    # Required. See "Permissions the caller must grant" below — without this
    # block the called workflow gets an empty token and cannot run at all.
    permissions:
      id-token: write # OIDC token to assume the AWS role
      contents: read # check out the Terraform code
      pull-requests: write # post the plan comment, react to /plan
    with:
      name: development
      working-directory: terraform/environments/development
      aws-role-arn: ${{ vars.AWS_ROLE_ARN }}

  production:
    uses: rockindahizzy/actions/.github/workflows/terraform-plan.yml@terraform-plan/v0
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    with:
      name: production
      working-directory: terraform/environments/production
      aws-role-arn: ${{ vars.PROD_AWS_ROLE_ARN }}
```

### Permissions the caller must grant

**These grants are mandatory, not hardening.** A called reusable workflow can
only ever *narrow* the calling workflow's `GITHUB_TOKEN` — it can never escalate
beyond it. The `permissions:` blocks inside `terraform-plan.yml` restrict what
each of its jobs may do with the token it is handed; they cannot add anything the
caller withheld.

So with `permissions: {}` at the top of your workflow and no job-level block, the
called workflow receives an **empty token** and fails at runtime:

- no OIDC token is issued, so `aws-auth` cannot assume the role, and
- the PR comment and reaction API calls are rejected.

`permissions: {}` at the top level is still the right default — it means a job
that does not declare its needs gets nothing. Declare them per calling job:

| Permission | Why |
|---|---|
| `id-token: write` | Issues the OIDC token used to assume the AWS role. |
| `contents: read` | Checks out the Terraform configuration. |
| `pull-requests: write` | Posts and updates the plan comment; reacts to `/plan`. |

Pair it with [`terraform-apply`](../terraform-apply/README.md) — the two are
designed together and the plan is useless without an apply to consume it.

The `issue_comment` trigger is required even for plan-only use, or `/plan` does
nothing.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Environment name. Identifies the environment in `/plan <name>`, in the artifact, and in the comment that gets updated. `[A-Za-z0-9._-]{1,64}`. Only the first **40** characters reach the AWS role session name, which AWS caps at 64 — longer names still work, but two environments sharing their first 40 characters are indistinguishable in CloudTrail. |
| `working-directory` | yes | — | Terraform root module for this environment. |
| `aws-role-arn` | yes | — | Role to assume via OIDC. Pass from `vars`, never hardcode. |
| `aws-region` | no | `us-west-2` | Region for the plan. |
| `terraform-version` | no | `latest` | Pin exactly in production — see below. |
| `var-file` | no | `""` | `-var-file` relative to `working-directory`. |
| `terraform-init-args` | no | `""` | Extra `terraform init` arguments. |
| `runs-on` | no | `ubuntu-latest` | Runner label. |
| `artifact-retention-days` | no | `5` | How long a plan stays applyable. |
| `comment-plan-output` | no | `true` | Post the plan body into the comment. Set `false` on public repositories — see [Secrets in plan output](#secrets-in-plan-output). |

### Outputs

`has-destroys`, `destroy-count`, `plan-artifact`.

### Pin `terraform-version`

The default is `latest`, which is convenient and wrong for anything real. A
saved plan is only reliably readable by the Terraform version that wrote it, so
with `latest` a release between plan and apply can leave a stored plan that no
longer opens. The apply workflow reads the actual version out of the plan
provenance and installs that exact version, which closes the window — but
pinning removes it entirely, and also stops Terraform upgrading underneath you
without a commit.

## Behaviour

- **On `pull_request`** — plans automatically on every push to a
  same-repository branch. **Fork PRs are not planned automatically** and report
  that a maintainer must review them first; see
  [Fork pull requests](#fork-pull-requests).
- **`/plan <name>`** — re-plans that one environment. This is also the only path
  by which a fork PR is ever planned, and requires write or admin.
- The result is posted as a PR comment, **updated in place** on each run rather
  than appended, keyed by a hidden marker per environment.
- **A failed plan is reported too.** If `terraform plan` fails — a provider auth
  error, a syntax error, a backend lock timeout, a missing `var-file` — the
  comment carries the tail of the plan output rather than leaving a red check
  with no explanation. Nothing is stored, so there is nothing to apply.
- The plan file is uploaded as an artifact for
  [`terraform-apply`](../terraform-apply/README.md) to consume.

Each caller job sees every `/plan` comment and exits quietly unless the command
names its own environment.

## The destroy guard

The check fails when the plan destroys **anything**, unless the PR has the
`allow-destroy` label.

Detection parses `terraform show -json tfplan` and counts `resource_changes`
whose `.change.actions` include `"delete"`. The human-readable plan output is
never grepped — its wording changes between Terraform versions and it is not an
interface.

**The guard fails closed.** Before counting anything it asserts that the document
parses as a JSON object and that `resource_changes` is present and is an array;
if not, the step fails loudly instead of reporting zero destroys. This matters
because jq's optional operator (`.resource_changes[]?`) silently yields `0` for a
document lacking that key — so a truncated write, a cross-version format change
or any unexpected shape would otherwise report "destroys nothing", mark the plan
applyable, and write `destroy_count: 0` into the provenance the apply job trusts.
An unreadable plan is a reason to refuse, not a reason to proceed.

**Replacements count.** A replacement is `["delete","create"]` or
`["create","delete"]`, and it destroys the resource just as thoroughly as a
plain destroy does. A guard that ignored replacements would miss the most common
way real infrastructure gets accidentally deleted — changing a `name` on a
resource that does not support in-place updates.

### The `allow-destroy` label

Create it once in the consuming repository:

```sh
gh label create allow-destroy \
  --description "Terraform plans on this PR may destroy resources" \
  --color d73a4a
```

Add it to a PR to permit destroys, then re-run `/plan <name>`. Labels are read
live from the API at check time, not from the event payload — the payload is
frozen at trigger time, so a label added in response to the failure would not
appear in it.

Because the label lives on the PR, it grants destroys for **every** environment
on that PR. Split destructive changes into their own pull request.

## Security

### Anyone can comment on a public repository

`issue_comment` fires for any GitHub user. Before doing anything, the workflow
calls `repos/{owner}/{repo}/collaborators/{user}/permission` and requires
`write` or `admin`. Anyone else gets a 👎 reaction and a comment saying why.

`author_association` from the comment payload is deliberately **not** used:
`CONTRIBUTOR` there means "has had a PR merged", not "has write access".

The permission check lives in a separate `context` job that runs to completion
before the job holding `id-token: write` starts. An unauthorised commenter never
reaches a step that can execute their Terraform.

### Checking out PR head from an `issue_comment` trigger

This is the part worth being explicit about.

`issue_comment` runs in the context of the **base** branch with the repository's
own `GITHUB_TOKEN`, which may have write access — unlike `pull_request` from a
fork, which is deliberately given a read-only token and no secrets. So checking
out PR head code under an `issue_comment` trigger deliberately re-couples two
things GitHub separates: **code the PR author controls** and **a privileged
token**.

That risk is accepted here, because it is unavoidable: planning a pull request
means running the pull request's Terraform. `terraform plan` is not a read-only
operation in the security sense — it runs provider code, `external` data
sources, and any `local-exec` in scope. A malicious PR can run arbitrary code on
the runner.

What limits the blast radius:

1. **Authorisation happens first, in a job with no AWS access.** Only someone
   with write access can trigger the run at all. The threat model is therefore a
   malicious *fork PR* whose code an authorised maintainer plans — not a drive-by
   commenter.
2. **`persist-credentials: false` on checkout.** The token is not left in
   `.git/config` where a `local-exec` could read it.
3. **The plan job never gets `contents: write`.** Code executing there cannot
   push to the repository.
4. **AWS access is the caller's to scope.** The OIDC role is the real privilege
   here. Scope it to what the environment genuinely needs and give production a
   different role from development.
5. **Fork pull requests are never planned automatically** — see below.

The residual risk is real and unavoidable: **review the diff before you type
`/plan` on a fork PR.** Planning is running code.

### Fork pull requests

**A fork PR never reaches AWS on the automatic path.** The `context` job compares
the head repository against the base repository and stops before the plan job —
which is the job holding `id-token: write` — is ever reached. The PR gets a
notice saying a maintainer must review it.

This gate is not optional decoration. `aws-role-arn` is a workflow **input**, not
a secret, so it is present on fork runs like any other; and OIDC is granted by the
plan job's own `permissions: id-token: write`, not withheld from forks by GitHub.
Nothing downstream would have stopped a fork PR from assuming the role. The
refusal has to happen before authentication, and it does.

> An earlier version of this document claimed that using `pull_request` rather
> than `pull_request_target` meant "fork PRs get a read-only token and no
> secrets". That was **wrong** as a mitigation here, for the two reasons above.
> The explicit head-repository comparison is what actually provides the
> protection.

**`pull_request_target` is refused outright.** It runs in base-branch context
with a privileged token and full secret access even for forks; combined with a
PR-head checkout and code execution it is the canonical full-compromise pattern.
The workflow fails with an explicit error if it is ever the triggering event, so
a consumer who wires it up finds out immediately. Use `on: pull_request`.

#### Can a maintainer `/plan` a fork PR? Yes — deliberately.

A `/plan <name>` comment from someone with write or admin access **will** plan a
fork PR, and this is a considered decision rather than an oversight:

- Refusing forks entirely would make the workflow useless for the open-source
  repositories it is aimed at, where nearly every contribution is a fork PR.
- The decision needs human judgement, and a maintainer who has read the diff is
  the only party able to supply it. The automatic path is refused precisely
  because no one has looked at the code yet.
- The comment path already requires write or admin, so the action is attributable
  to a named person and is recorded in the run log.

What this is **not** is a safety guarantee. Planning fork code executes that code
with the environment's AWS role, so the maintainer typing the command is
accepting exactly that risk. The run emits a warning naming the fork and the
requester, so the decision is auditable afterwards.

**Read the diff before you type `/plan` on a fork PR** — with particular
attention to `external` data sources, `local-exec` provisioners, and any
unfamiliar module or provider source.

### Secrets in plan output

Terraform renders every attribute a provider did not mark `sensitive`. That
routinely includes connection strings, generated passwords surfaced through a
non-sensitive output, and anything read from a data source.

- The plan body posted into the PR comment is truncated to 50000 characters, and
  can be turned off entirely with `comment-plan-output: false`. **On a public
  repository, set it to `false`** — a PR comment is world-readable, while the
  job log requires read access to the repository.
- `plan.json` is **never uploaded**. It contains every attribute in structured
  form, and artifacts are downloadable by anyone who can read the repository.
  Only the binary `tfplan` is stored.

Neither measure makes plan output safe to publish. They reduce how far it
travels by default.
