# private-git-dep

Configures git so `yarn install` (or npm, or pnpm) can resolve git dependencies
that live in a **private** repository.

The default `GITHUB_TOKEN` is scoped to the one repository running the workflow.
A `package.json` that depends on a second private repo therefore fails at install
time. This action supplies a caller-provided credential and rewrites GitHub
remotes to authenticated HTTPS so the fetch succeeds.

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

- uses: rockindahizzy/actions/private-git-dep@private-git-dep/v0
  with:
    token: ${{ secrets.DS_TOKEN }}

- run: yarn install --frozen-lockfile
```

## The ssh detail, which is the whole point

The obvious one-liner for this problem is:

```bash
# Does NOT fix the shorthand form.
git config --global url."https://x-access-token:${TOKEN}@github.com/".insteadOf "https://github.com/"
```

It rewrites `https://`. But the dependency form that motivates this action —

```json
"articulate-llm-design": "ArticulateLLM/articulate-llm-design#v0.3.2"
```

— is not fetched over https. npm and Yarn expand the `Owner/repo#ref` shorthand
through `hosted-git-info`, whose default representation for a shortcut is **ssh**.
Yarn runs:

```
git ls-remote --tags --heads ssh://git@github.com/ArticulateLLM/articulate-llm-design.git
```

An https-only rewrite never matches that, and the install fails with
`Permission denied (publickey)` — with a token correctly configured, which makes
it a genuinely confusing failure.

This action rewrites every form GitHub remotes appear in: `https://`,
`ssh://git@`, `git+ssh://git@`, `git://`, and the scp-like `git@host:`. The
self-test asserts on all five, and separately asserts that a real `yarn install`
switches from ssh to https once the action has run.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `token` | yes | — | Token with read access to the private repositories. |
| `username` | no | `x-access-token` | Basic-auth username paired with the token. |
| `host` | no | `github.com` | Git host to rewrite. Change for GHES. |
| `scope` | no | `local` | `local` (repo `.git/config`) or `global` (`~/.gitconfig`). |
| `working-directory` | no | `.` | Which repository receives a `local` rewrite. |
| `cleanup` | no | `true` | Warn and expose the section to remove, when `scope: global`. |

## Which credential

In preference order:

**A GitHub App installation token.** Scoped to chosen repositories, expires after
one hour, and rotates itself. Nothing long-lived sits in a secret store.

```yaml
- uses: actions/create-github-app-token@<sha> # v3.2.0
  id: app-token
  with:
    app-id: ${{ vars.DS_APP_ID }}
    private-key: ${{ secrets.DS_APP_PRIVATE_KEY }}
    owner: ${{ github.repository_owner }}
    repositories: articulate-llm-design

- uses: rockindahizzy/actions/private-git-dep@private-git-dep/v0
  with:
    token: ${{ steps.app-token.outputs.token }}
```

The cost is setup: an App must be created, installed, and its private key stored.
Note the App's private key is itself long-lived — the rotation benefit is real but
it moves the secret rather than eliminating it.

**A fine-grained PAT** with `Contents: read` on exactly the needed repositories.
Faster to set up, and the reasonable choice to get unblocked. It expires on a
fixed date, so it will fail in CI on a day nobody chose — set a calendar reminder.
Fine-grained PATs are user-owned, so the access disappears if that user loses
access to the org; a machine account avoids tying CI to a person.

**Never a classic PAT.** Its scopes cannot be narrowed below "every repository the
user can read". In a public repository's CI, that is the whole account.

This answers the open question in issue #2: an App is the right destination, a
fine-grained PAT on a machine account is an acceptable interim.

## Scope, and self-hosted runners

`scope: local` (the default) writes to the current repository's `.git/config`.
The credential lives inside the workspace and cannot leak into another job. This
is the important default for **self-hosted runners**, where `~/.gitconfig`
persists between jobs — a global rewrite there leaves a working credential on
disk for whatever runs next, including a workflow from a different repository.

Use `scope: global` only when dependency resolution runs outside this repository
(a build in a temp directory, a tool that clones elsewhere). On a GitHub-hosted
runner the VM is destroyed after the job, so the exposure ends with it. On a
self-hosted runner it does not.

### Composite actions cannot clean up after themselves

`post:` steps are only available to JavaScript and Docker actions. A composite
action has no way to register a teardown that runs at job end — this is
[actions/runner#1478](https://github.com/actions/runner/issues/1478), open since
2021. So with `scope: global` on a self-hosted runner, **you must clean up**:

```yaml
- uses: rockindahizzy/actions/private-git-dep@private-git-dep/v0
  with:
    token: ${{ secrets.DS_TOKEN }}
    scope: global

- run: yarn install --frozen-lockfile

- name: Remove the credential
  if: always()      # must run even when the build fails
  run: |
    if [ -n "${PRIVATE_GIT_DEP_SECTION:-}" ]; then
      git config --global --remove-section "$PRIVATE_GIT_DEP_SECTION" || true
    fi
```

The action exports `PRIVATE_GIT_DEP_SECTION` for this. The self-test runs exactly
this snippet and asserts `~/.gitconfig` is clean afterwards, so the documented
cleanup is verified rather than merely described.

With the default `scope: local`, no cleanup step is needed.

## The rewrite is broad by design

It applies to **every** `github.com` fetch for the rest of the job (or the
lifetime of the config), not only the intended dependency. Any `git clone` of a
public GitHub repo in a later step also sends the token. That is inherent to
`insteadOf` — it matches on host, and cannot be narrowed to one repository.

The mitigation is the credential, not the rewrite: scope the App installation or
fine-grained PAT to only the repositories it needs, and a leak elsewhere grants
nothing extra. Rewriting is scoped to `host`, so other hosts are untouched.

## Handling of the credential

- Masked with `::add-mask::` before anything else runs. Masking is best-effort:
  it matches the literal string only, so a transformed copy (base64, URL-encoded,
  split across lines) is not redacted. The real protection is not printing it.
- Passed into the step as an environment variable, never a `${{ }}` expansion
  inside the script body. An expansion is substituted into the script *text*,
  which would place the credential in a file on disk and in any shell trace.
- **Not** written into any remote's URL. `insteadOf` rewrites at fetch time and
  leaves `.git/config` remotes as they are, so the token does not reach
  `.git/config` via a remote, nor a lockfile, nor a commit.

One sharp edge worth knowing: because the rewrite applies at resolution time,
`git remote -v` and `git remote get-url` **print the token in cleartext**. They
are safe to run but must never be echoed into a log. For the same reason the
action prints only the left-hand side of its config — `git config --get-regexp`
output contains the token in the section name.

## What the self-test does not cover

A public repository has no private-repo credential, so the final hop — a real
token fetching a real private dependency — cannot be exercised here. The self-test
covers everything up to that boundary: the rewrite is written for all five
protocols, a real `yarn install` changes protocol because of it, the token stays
out of printed output, `local` never touches `~/.gitconfig`, rotation replaces
the old credential, cleanup empties the global config, and bad input fails closed.

What remains unverified is that a valid token is accepted by GitHub — which is a
property of the token, not of this action. Verify that once in a consuming
repository, per the last checkbox on issue #2.
