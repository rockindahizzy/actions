# 1. Per-action semantic versioning, driven by conventional commits

Date: 2026-08-08

## Status

Accepted

Resolves [#6](https://github.com/rockindahizzy/actions/issues/6). Coupled with
[#5](https://github.com/rockindahizzy/actions/issues/5), which owns the
SHA-pinning guidance this decision depends on.

## Context

This repository publishes shared GitHub Actions consumed by personal repositories
and five organizations. Consumers reference actions by git ref, so the tagging
scheme is frozen the moment anything is published — changing it later means
editing every consuming workflow.

Nothing has been published yet. No tags exist, and no actions are built. This is
the last point at which the convention is free to choose.

Two things forced the decision now rather than later:

- Issues #1–#4 each create an action that needs a ref to publish under.
- The README already documented `@v1` repo-wide, as though the question were
  settled. Leaving that unexamined would have made it true by default.

The repository will hold two artifact shapes, and they are not symmetric:

- **Composite actions** — `<action-name>/action.yml`, self-contained directories.
- **Reusable workflows** — `.github/workflows/<name>.yml`, flat files. GitHub
  does not permit subdirectories there, so they have no directory of their own to
  version against.

## Decision

### Per-action versions, not repo-wide

Each action carries its own `MAJOR.MINOR.PATCH`, independent of every other
action in the repository.

### Tag format: `<action>/v0.x.x`

Slash separator, `v` prefix, component name first.

```yaml
uses: rockindahizzy/actions/aws-auth@aws-auth/v0.2.1   # exact
uses: rockindahizzy/actions/aws-auth@aws-auth/v0       # moving
```

A consequence: **no bare tag may ever be named after an action.** Git cannot hold
both a tag `aws-auth` and tags under `aws-auth/`.

### Start at v0, with pre-1.0 semantics

Actions publish as `v0` and stay there while their interface settles. Under
`0.x`, a breaking change bumps the minor (`0.1.0 → 0.2.0`) and a feature bumps
the patch.

The moving tag is the major — `<action>/v0` — consistent with how `v1` will
behave later. Breaking changes therefore *do* reach consumers pinned to `v0`.
That is what `v0` communicates, and it is why SHA pinning matters more during
this phase rather than less.

### Versions are computed, never chosen

Bumps derive from conventional commit messages scoped to the action whose files
changed. No human picks a version number.

Tooling is [release-please](https://github.com/googleapis/release-please):

```jsonc
{
  "tag-separator": "/",
  "include-component-in-tag": true,
  "include-v-in-tag": true,
  "bump-minor-pre-major": true,
  "packages": {
    "aws-auth": { "release-type": "simple" }
  }
}
```

`release-type: simple` suits directories with no package manifest.
`bump-minor-pre-major` is what makes `v0` meaningful — without it the first
breaking change jumps straight to `1.0.0`.

**release-please does not move major tags.** It creates the exact version tag and
the GitHub release, and stops. Moving `<action>/v0` is a second step, chained off
the `autorelease: tagged` state.

### Reusable workflows version through a thin directory

Reusable workflows must live flat in `.github/workflows/` — subdirectories are
not supported — and release-please can only version directories, never single
files. The two constraints are incompatible.

Each reusable workflow therefore gets a directory that acts as its versioned
component, alongside the callable file:

```
terraform-plan/               versioned component -> terraform-plan/v0.1.0
.github/workflows/
  terraform-plan.yml          the file consumers call
```

Consumers reference it as
`...github/workflows/terraform-plan.yml@terraform-plan/v0.1.0`.

Every artifact in the repository then versions identically, and consumers learn
one rule rather than two. The cost is that nothing inherently keeps the
component and the callable file in step, so **CI must check they agree.**

### Release PRs are combined

`separate-pull-requests: false`. One release PR covers every action with pending
changes, and merging it releases them all. One decision point rather than
several. The trade is that one action cannot be released while another is held
back.

### Conventional commits are enforced on PR titles

A CI check validates the PR title against the conventional format, and merges
are squashed so that title becomes the commit message. Only one string per pull
request has to be correct, and it is visible and editable before merge.

Automatic versioning depends entirely on this discipline: a malformed message
means a release silently does not happen, with no error to notice.

### Pinned SHAs are refreshed by Dependabot

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directories:
      - "/"        # .github/workflows
      - "**/*"     # composite action.yml files
    schedule:
      interval: weekly
    groups:
      actions:
        patterns: ["*"]
```

The `directories` glob is not optional. Dependabot's default configuration scans
only `.github/workflows/`, so actions referenced inside a composite `action.yml`
would never be updated — including the `aws-actions/configure-aws-credentials`
pin inside `aws-auth`.

Pins carry a version comment (`@a1b2c3d # v4.2.1`); without one the hash cannot
be resolved to a release.

No automerge. An automatically merged dependency update is a direct path into a
repository whose actions hold OIDC access to AWS.

### Trunk-based, no LTS

Single `main`. Short-lived branches merge to it; release-please maintains a
standing release PR, and merging that PR cuts the release. The human gate is
"ship what has accumulated?", not "what version?".

No maintenance branches and no backports. When an action reaches a new major,
consumers move forward; the previous major receives nothing, including security
fixes.

### Dogfooding

1. **Every third-party action is SHA-pinned**, per the rule this repository asks
   consumers to follow.
2. **This repository's CI consumes its own actions**, via `$/` at the running
   commit. `release.yml` is the exception and references nothing from this
   repository, so a broken action can never block the release that fixes it.
   Breakage still fails ordinary CI — that is the signal dogfooding exists to
   produce.
3. **Consumer repositories migrate at v0**, accepting churn in exchange for real
   usage surfacing real problems.

### Composition uses `$/`, never `./`

Actions in this repository may reference each other, using the self-repository
syntax:

```yaml
# aws-auth/action.yml
- uses: $/setup-node-yarn
```

`$/` resolves against the repository and SHA of the file containing it. A
consumer calling `aws-auth@aws-auth/v0.2.1` gets the `setup-node-yarn` at that
same commit. There is no version to drift and no ref for release-please to keep
current.

Workspace-relative `./` must not be used for this. Inside a composite action it
resolves against the *consumer's* checkout rather than the action's own
directory, so it only works if the consumer happens to have checked this
repository out at a matching path.

`$/` requires runner 2.336.0 or newer. Confirm this across consuming
organizations before relying on it, particularly any running self-hosted
runners.

### Verification is by execution, not static analysis

Each action gets a self-test workflow that invokes it through `$/` with real
inputs and asserts on the result. These run on GitHub's own runners, so what is
verified is what consumers get.

The failures that matter here are behavioural — an action assuming an input
nobody sets, or breaking against a new runtime version. No linter finds those.
Because self-tests use `$/`, they exercise the action as changed in the pull
request, before any tag exists: testing and dogfooding are the same mechanism.

`aws-auth` is testable end to end. It needs a trust policy scoped to
`repo:rockindahizzy/actions:*` and a test role with **no policies attached** —
`aws sts get-caller-identity` requires no permissions and proves the whole chain
(token issued, trust matched, role assumed, credentials exported). A compromise
of this public repository would yield a session that can identify itself and
nothing else.

The trust policy cannot be restricted to `refs/heads/main`, since self-tests run
on pull requests.

### Internal actions are not published

Actions used only by this repository's CI — release plumbing, lint runners, the
hardcoded-ARN check — live here untagged and unversioned. release-please
versions only what its `packages` config names, so exclusion is structural
rather than a convention someone must remember.

## Consequences

### Good

- A breaking change to one action does not force a version bump on consumers of
  every other action.
- Version numbers cannot drift from what the commits actually say.
- Starting at `v0` sets the honest expectation that these interfaces will move.
- Dogfooding means this repository experiences its own guidance before consumers
  do.

### Bad

- Consumers track several versions rather than one. Accepted: the alternative is
  spurious major bumps for changes that did not touch the action they use.
- Releases depend on commit-message discipline. A malformed message means a
  release silently does not happen. Wants a CI check.
- The moving `v0` tag carries breaking changes to anyone pinned to it. Mitigated
  by #5's rule: pin by SHA wherever the calling workflow holds elevated
  permissions.
- No LTS means a consumer on an old major has no upgrade path but forward. For a
  repository this size, supporting old majors is a commitment we decline to make.
- Two moving parts in the release pipeline — release-please plus a major-tag
  mover — rather than one tool.
- SHA pinning without a refresh bot trades a supply-chain risk for a
  stale-dependency one.
- Composition depends on `$/`, which became generally available on 2026-07-30
  and requires runner 2.336.0. Any consumer on an older self-hosted runner
  cannot use a composed action.
- Reusable workflows carry a directory that exists only to satisfy the release
  tooling, and a CI check to keep it honest. Consistency is bought with a piece
  of scaffolding that will look pointless to anyone who does not know why.
- A combined release PR means one action cannot ship while another waits.
- Squash-merging discards individual commit messages within a pull request.
- Self-tests need a real AWS test role, adding Terraform work to #1 beyond what
  that issue currently describes.

### Unresolved

Construction details only: which major-tag mover to use, what a thin workflow
directory contains, where unpublished internal actions live, and whether
actionlint is worth adding for the workflow files.

## Alternatives considered

**Repo-wide semver with a moving major** — the ecosystem norm, and what the
README originally assumed. Rejected: `actions/checkout` and similar hold one
action per repository, so the multi-action case never arises for them. Here, a
breaking change to the Terraform workflow would bump `aws-auth` to a new major
for a change that never touched it. With four actions planned and five consuming
organizations, that is the ordinary case within a year.

**semantic-release-monorepo** — namespaces tags per package. Rejected: it cannot
release several packages in one run. After the first sub-release it writes a tag,
and the next run finds no changes. A PR touching two actions would half-release.

**multi-semantic-release** — addresses that limitation, but is self-described as
a proof of concept.

**Manual or dispatch-triggered versioning** — rejected outright. Version numbers
chosen by hand drift from what changed.

**Forbidding composition between actions in this repository** — considered, and
correct until very recently. Composition would have required an outer action to
pin an inner one by tag; release-please does not update those refs, so an inner
bump would have left the outer action silently stale.

`$/` removes the problem rather than mitigating it: because it resolves to the
same repository and commit as the file containing it, there is no ref to keep
current. Worth recording that this reasoning is only nine days old — anyone
revisiting this decision should check whether `$/` still behaves as described.

**`act` as the verification gate** — rejected. It cannot issue OIDC tokens (only
GitHub can sign for `token.actions.githubusercontent.com`), so it cannot test
`aws-auth` at all; its runner images differ from GitHub-hosted ones; and
composite secret isolation behaves differently. Testing against a different
environment than consumers use is weak verification. `act` remains useful for
local iteration on non-AWS actions.

**Static analysis as the primary check** — rejected as the main strategy.
actionlint does not lint composite action definition files, so in a repository
whose main product is composite actions it checks everything except the primary
artifacts. Useful for workflow files; not a substitute for execution.

**Renovate for dependency updates** — rejected in favour of Dependabot. Its
advantage was collapsing many pin updates into one pull request; Dependabot
gained cross-directory grouping in February 2026, which closes that gap at this
scale. Renovate would add a third-party application with write access to a
repository five organizations depend on, to save a few pull requests a month.

**Grouping all reusable workflows under one version** — simpler, and it avoids
the phantom directories. Rejected because a change to `terraform-apply` would
bump `terraform-plan`, reintroducing in miniature the coupling this ADR exists
to remove.
