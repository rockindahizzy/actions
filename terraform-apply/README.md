# terraform-apply

Applies the plan stored by [`terraform-plan`](../terraform-plan/README.md), on
the pull request, triggered by `/apply <name>` from someone with write access.

The workflow file consumers call is
[`.github/workflows/terraform-apply.yml`](../.github/workflows/terraform-apply.yml).

## Why this directory exists

It is the versioned component for that workflow and contains no executable code.
Reusable workflows must live flat in `.github/workflows/` while release-please
can only version directories — see
[`terraform-plan/README.md`](../terraform-plan/README.md#why-this-directory-exists)
and [ADR 0001](../docs/adr/0001-per-action-semantic-versioning.md).

## Apply happens on the PR, never after merge

The Atlantis model. `terraform apply` is the step that changes the world, so it
runs while the plan is still on screen and while backing out is still just "do
not merge". Applying after merge means `main` claims a change is live before
anyone has confirmed that it worked, and a failed post-merge apply leaves the
default branch describing infrastructure that does not exist.

**There is no apply-after-merge path and no other apply path.** The workflow
triggers on `issue_comment` only, and refuses outright if the PR is closed or
merged.

## Usage

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
    # Required. See "Permissions the caller must grant" below.
    permissions:
      id-token: write # OIDC token to assume the AWS role
      contents: read # check out the Terraform code
      pull-requests: write # post the plan comment, react to /plan
    with:
      name: development
      working-directory: terraform/environments/development
      aws-role-arn: ${{ vars.AWS_ROLE_ARN }}

  development-apply:
    uses: rockindahizzy/actions/.github/workflows/terraform-apply.yml@terraform-apply/v0
    # The apply needs two more than the plan: `statuses: write` to emit the
    # commit status your ruleset gates merge on, and `actions: read` to find
    # and download the stored plan artifact from the plan run.
    permissions:
      id-token: write # OIDC token to assume the AWS role
      contents: read # check out the Terraform code
      pull-requests: write # post the result comment, react to /apply
      statuses: write # emit the terraform-apply/<name> commit status
      actions: read # locate and download the stored plan artifact
    with:
      name: development
      working-directory: terraform/environments/development
      aws-role-arn: ${{ vars.AWS_ROLE_ARN }}
```

### Permissions the caller must grant

**These grants are mandatory, not hardening.** A called reusable workflow can
only *narrow* the calling workflow's `GITHUB_TOKEN`, never escalate past it. The
`permissions:` blocks inside `terraform-apply.yml` constrain what its own jobs do
with the token they are handed; they cannot add anything the caller withheld.

With `permissions: {}` at the top and no job-level block, the called workflow
gets an **empty token**: OIDC issuance fails so the AWS role cannot be assumed,
the artifact lookup returns nothing, the commit status is never written — so
merge stays blocked forever — and the result comment cannot be posted.

Keep `permissions: {}` at the top level as the default-deny, and declare per job:

| Permission | Plan | Apply | Why |
|---|:--:|:--:|---|
| `id-token: write` | ✓ | ✓ | Issues the OIDC token used to assume the AWS role. |
| `contents: read` | ✓ | ✓ | Checks out the Terraform configuration. |
| `pull-requests: write` | ✓ | ✓ | Posts comments and reactions on the PR. |
| `statuses: write` | | ✓ | Emits the `terraform-apply/<name>` status your ruleset requires. |
| `actions: read` | | ✓ | Reads the artifacts and workflow-run APIs to find the stored plan and verify it came from `terraform-plan`. |

If you gate merge on the status check, `statuses: write` is what makes the gate
work at all — without it the status is never reported and the PR cannot merge.

`name` and `working-directory` **must match the plan job's** — a mismatch is
detected and refused rather than applied.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Environment name. Must match the plan job. |
| `working-directory` | yes | — | Must match the plan job. |
| `aws-role-arn` | yes | — | Role to assume via OIDC. |
| `aws-region` | no | `us-west-2` | Region for the apply. |
| `terraform-init-args` | no | `""` | Extra `terraform init` arguments. |
| `runs-on` | no | `ubuntu-latest` | Runner label. |
| `status-check-name` | no | `terraform-apply/<name>` | Context of the emitted commit status. This is the string your ruleset needs. |
| `environment` | no | `""` | GitHub Environment for the apply job — the supported way to add a manual approval gate. |

Output: `applied`.

The Terraform version is **not** an input. It is read from the plan's provenance
record and the exact version that wrote the plan is installed, because a saved
plan is only reliably readable by the version that produced it.

## Gating merge on the apply — the ruleset you need

The workflow emits a **commit status** named `terraform-apply/<name>` (or your
`status-check-name`) against the PR head:

| State | Meaning |
|---|---|
| `pending` | apply in flight |
| `success` | applied cleanly |
| `failure` | refused, or the apply failed |

Whether merge is *blocked* on that status is your repository's decision, not
this workflow's. To get true Atlantis behaviour — nothing merges until it has
been applied — add a ruleset in the consuming repository:

**Settings → Rules → Rulesets → New branch ruleset**

- Target: the default branch
- Enable **Require status checks to pass**
  - Add `terraform-apply/development` (one entry per environment)
  - Enable **Require branches to be up to date before merging**
- Enable **Require a pull request before merging**

Or via the API:

```sh
gh api --method POST repos/{owner}/{repo}/rulesets \
  --input ruleset.json
```

```json
{
  "name": "terraform-applied-before-merge",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] }
  },
  "rules": [
    { "type": "pull_request" },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "terraform-apply/development" },
          { "context": "terraform-apply/production" }
        ]
      }
    }
  ]
}
```

Two things to know:

- **A status that has never been reported reads as pending, which blocks merge.**
  That is the desired behaviour — a PR nobody applied should not merge — but it
  means every environment listed in the ruleset must be applied on every PR,
  including PRs that change no Terraform at all. If that is too strict, list
  only the environments that genuinely gate merge.
- **`strict_required_status_checks_policy` (require branches up to date)
  matters.** Without it, a PR applied at commit A can merge after main has moved
  on. With it, the branch must be current — and because a new commit invalidates
  the stored plan (below), it must be re-planned and re-applied.

## Staleness: two different checks

**Terraform's own, which is not reimplemented here.** A saved plan records the
state's `lineage` and `serial`. If the state moved after the plan was made,
`terraform apply tfplan` refuses with `Error: Saved plan is stale`. This is
stricter than any commit comparison because it tracks the state rather than the
code — two PRs planned in parallel, the first applied, and the second is refused
automatically. There is no re-plan-and-compare here; that would be a weaker
version of a check Terraform already performs.

**The commit check, which Terraform cannot do.** A plan made at commit A applies
perfectly at commit B as long as the state never moved — nothing in the plan
file records the commit. So the reviewed plan for old code would go into effect
while the PR displays new code.

The plan run therefore stores a provenance record alongside the plan, and the
apply refuses when its `head_sha` no longer matches the PR head. **The head SHA
is re-read from the API inside the check itself**, not carried over from the
authorisation job: jobs are scheduled separately, and a push landing in the gap
between them would otherwise be invisible — the comparison would be plan-SHA
against a stale snapshot rather than against the real current head. The freshly
read value is what the checkout and the commit status use too, so all three refer
to the same commit.

> The stored plan for `development` was made against commit `a1b2c3d`, but this
> pull request now points at `e4f5a6b`. […] Run `/plan development` to plan the
> current commit, then `/apply development`.

An apply is also refused when: no plan is stored (never planned, or the artifact
expired), the provenance is missing, the environment or working directory
disagrees with the plan, or the plan destroys resources and the `allow-destroy`
label is absent.

### Finding the plan across runs

The plan is uploaded by the `terraform-plan` run and consumed by a different
`terraform-apply` run. `actions/download-artifact` defaults `run-id` to the
*current* run, so the workflow first queries the artifacts API to find the run
holding the plan and downloads from that run explicitly. Artifact names are
unique per run rather than per repository, so several runs can hold one — every
push that re-planned.

**The artifact's name is treated as a search filter, never as proof of origin.**
Every downstream gate reads its inputs out of that artifact: the destroy re-check
takes `destroy_count` from `plan-meta.json`, the commit check takes `head_sha`,
the Terraform version comes from it, and the binary `tfplan` that is applied *is*
the artifact. Artifact names are entirely predictable (`tfplan-<pr>-<env>`), and
any workflow run in the repository can upload an artifact under any name — so
selecting on name alone would let a lookalike declaring `destroy_count: 0` and a
matching `head_sha` sail through every gate and be applied.

A candidate is therefore accepted only when its producing run:

- is the **`terraform-plan` workflow** of this repository, identified by the
  run's `path` (`.github/workflows/terraform-plan.yml`) — assigned by GitHub from
  the workflow file that actually ran, so a run cannot claim it by naming itself
  `terraform-plan`;
- has a **`head_sha` equal to the pull request's current head**; and
- **is not from a fork** (`head_repository_id == repository_id`).

If artifacts of the right name exist but none passes, the apply is refused with a
message distinguishing that from "no plan at all", because the causes and the
remedies differ — and because silently reporting "no stored plan" would hide a
possible substitution attempt.

The practical consequence: **a plan lives exactly as long as its artifact.**
Once `artifact-retention-days` (default 5) elapses, `/apply` refuses with "no
stored plan" and the environment must be re-planned.

Every refusal posts a comment saying exactly which check failed, and **nothing
is applied**.

## The destroy guard is re-checked at apply time

The plan run already blocks destroys without the `allow-destroy` label, but the
label can be removed between plan and apply. The apply re-reads the destroy
count from the plan provenance and re-checks the label live before touching AWS.

**It fails closed.** A provenance record that is unreadable, or whose
`destroy_count` is missing, null or non-numeric, is refused rather than treated
as "destroys nothing" — the most dangerous possible default for the last gate
before a real destroy. The same rule applies in the plan workflow, where the
change counts are only trusted once the plan JSON has been asserted to contain a
`resource_changes` array.

## Security

### Authorisation

`/apply` requires `write` or `admin` via
`repos/{owner}/{repo}/collaborators/{user}/permission`. Anyone else gets a 👎 and
a comment. The check runs in an `authorize` job that holds **no AWS permission at
all**; the job with `id-token: write` does not start until it passes.

Without this, a stranger commenting `/apply production` on a public repository
triggers a real apply against real infrastructure.

`author_association` is not used — `CONTRIBUTOR` means "has had a PR merged".

### What the reactions mean

| Reaction | Meaning |
|---|---|
| 👀 | Command accepted and the commenter is authorised. Every apply-side gate is still ahead. |
| 🚀 | All gates passed; `terraform apply` is starting. |
| 👎 | Refused — the commenter lacks write access. |

The two are distinct because refusals after authorisation are common — no stored
plan, a moved commit, a missing label — and a 🚀 at authorisation time would read
as "apply started" on runs where nothing is ever applied.

### The command must be the first line

Only the first non-blank line of a comment is parsed, so quoting someone else's
`/apply production` in a reply cannot trigger an apply. Extra words are rejected
rather than ignored.

### Checking out PR head code

The apply checks out the same commit the plan was made against, having already
proven it equals the current PR head. The reasoning about `issue_comment`,
privileged tokens and PR-authored code is the same as for the plan workflow and
is written out in
[`terraform-plan/README.md`](../terraform-plan/README.md#checking-out-pr-head-from-an-issue_comment-trigger).

Apply is the more dangerous of the two, and the mitigations are the same ones,
plus: authorisation is required before any download or checkout happens,
`persist-credentials: false` keeps the token out of `.git/config`, and the job
never holds `contents: write`.

Set `environment:` to add a required-reviewers gate on top, for production.

### Apply output is never posted to the PR

Terraform echoes resource attributes as they are created, which is the output
most likely to contain something a provider did not mark `sensitive`. The
success comment links to the job log instead — readable only by users with
access to the Actions tab, rather than by everyone on the internet.

### Concurrency

`cancel-in-progress: false`, grouped per environment. An apply in flight is
never cancelled — a half-applied Terraform run is the worst available outcome —
and two applies of the same environment never overlap. This is the opposite of
the plan workflow, where cancelling a superseded plan costs nothing.

## Commands

| Comment | Effect |
|---|---|
| `/plan <name>` | Re-plan that environment |
| `/apply <name>` | Apply that environment's stored plan |

Both require write or admin. Both target exactly one environment; there is no
"all environments" form, deliberately.
