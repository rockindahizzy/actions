# setup-node-yarn

Node.js plus Yarn 4 Berry via Corepack, with the Yarn cache restored and an
optional immutable install.

The Yarn version comes from the `packageManager` field in your `package.json`.
This action never pins one — see [Why no Yarn version](#why-this-action-does-not-pin-a-yarn-version).

## Usage

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      - uses: rockindahizzy/actions/setup-node-yarn@setup-node-yarn/v0
        with:
          node-version: 20
          install: true

      - run: yarn test
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `node-version` | `20` | Node version. Anything `actions/setup-node` accepts: `20`, `20.11.1`, `20.x`. |
| `install` | `true` | Run `yarn install --immutable`. |
| `working-directory` | `.` | Directory holding `package.json` and `yarn.lock`. |
| `checkout` | `false` | Have the action check out the repository. See below. |
| `cache` | `true` | Save and restore the Yarn cache. |
| `cache-key-prefix` | `setup-node-yarn` | Cache key prefix; change it to segment or invalidate caches. |

## Outputs

| Output | Description |
|---|---|
| `node-version` | Active Node version, e.g. `v20.19.0`. |
| `yarn-version` | Yarn version Corepack resolved, e.g. `4.5.3`. |
| `cache-hit` | `true` on an exact cache key match; empty when caching is off. |

## Does this action check out your repository?

**No, by default — you check out, then call this action.** The example above
shows the intended shape.

The issue that requested this action sketched it without a checkout step, so
this was a real decision rather than an oversight. A composite action *can*
call `actions/checkout`, but it can only ever check out the **default
repository at the triggering ref**. That is correct for the simple case and
quietly wrong for several common ones:

- `pull_request_target`, where the default ref is the base branch and checking
  out the PR head is a deliberate, security-sensitive choice
- workflows needing submodules, LFS, or a non-default `fetch-depth`
- monorepos checking out a different repository, or several
- any job that has already checked out — a second checkout wipes the working
  tree, discarding generated files

Doing it unconditionally would trade a saved line for a class of failures that
are awkward to debug. Leaving it to the caller keeps checkout configuration
where the caller can see it.

There is an escape hatch. If a job needs nothing but this action, set
`checkout: true`:

```yaml
- uses: rockindahizzy/actions/setup-node-yarn@setup-node-yarn/v0
  with:
    checkout: true
```

This is a convenience for the simple case, not the recommended default.

## Why this action does not pin a Yarn version

`articulate-llm` and `articulate-llm-web` both declare their Yarn version in
`package.json`:

```json
{ "packageManager": "yarn@4.5.3" }
```

That field is the single source of truth. Local development, `yarn` on a
laptop, and CI all read it, so pinning a version here would create a second
place to change and a way for CI to disagree with developer machines.

The action enables Corepack and lets your repository decide. It fails with a
clear message if `packageManager` is missing, rather than silently falling back
to whatever Yarn the runner happens to have.

## Caching

The cache path is **resolved at runtime**, not hardcoded, because Yarn Berry's
cache location depends on your configuration:

- Yarn 4 defaults `enableGlobalCache` to `true`, putting the cache in
  `~/.yarn/berry/cache`
- setting `enableGlobalCache: false` moves it to `.yarn/cache` inside the
  project

Many guides still assume the project-local path. Hardcoding either one caches
the wrong directory and never hits, which fails silently as a slow build rather
than a broken one. The action asks Yarn (`yarn config get cacheFolder`) and
caches what it reports.

The cache key combines the OS, the Node version and a hash of `yarn.lock`. Node
and OS are in the key because the cache holds build output for packages with
native components, which is not portable between them. A `restore-keys` prefix
lets a changed lockfile start from the nearest previous cache and download only
what differs.

### `install: false` still sets up the cache

Cache setup is independent of installing. A job that skips the install but runs
`yarn` later still gets a warm cache, and a job that installs a subset of
workspaces itself does not pay a full download. To turn caching off, set
`cache: false` explicitly.

## Immutable installs

`install: true` runs `yarn install --immutable`, which **fails** rather than
rewriting `yarn.lock`. That is deliberate: it makes CI catch a `package.json`
change committed without its lockfile. If your install fails with a lockfile
error, run `yarn install` locally and commit the result.

## Examples

Toolchain only, no install:

```yaml
- uses: rockindahizzy/actions/setup-node-yarn@setup-node-yarn/v0
  with:
    install: false
```

Matrix over Node versions:

```yaml
strategy:
  matrix:
    node-version: [20, 22]
steps:
  - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
  - uses: rockindahizzy/actions/setup-node-yarn@setup-node-yarn/v0
    with:
      node-version: ${{ matrix.node-version }}
```

A Yarn project in a subdirectory:

```yaml
- uses: rockindahizzy/actions/setup-node-yarn@setup-node-yarn/v0
  with:
    working-directory: packages/web
```

## Pinning

`setup-node-yarn/v0` is a moving tag that tracks the latest `0.x`. Under `0.x` a
breaking change bumps the minor, so **`v0` will carry breaking changes**. Pin to
an exact version (`setup-node-yarn/v0.1.0`) or a commit SHA if that is not
acceptable. See [ADR 0001](../docs/adr/0001-per-action-semantic-versioning.md).
