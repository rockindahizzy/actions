# Releasing

How this repository turns merged commits into published action versions. The
convention itself is [ADR 0001](adr/0001-per-action-semantic-versioning.md);
this is the mechanism.

## Adding an action to the release process

Add an entry to `release-please-config.json`:

```json
"packages": {
  "aws-auth": {
    "release-type": "simple",
    "component": "aws-auth",
    "initial-version": "0.1.0"
  }
}
```

**`component` is required, despite `include-component-in-tag` being set
globally.** With `release-type: simple` there is no package manifest to read a
name from, so `packageName` is undefined and release-please falls back to
`normalizeComponent(undefined)`, which returns an empty string
(`src/strategies/base.ts`). `include-component-in-tag: true` then faithfully
includes nothing, and the tag comes out as a bare `v0.1.0`.

With two such packages the failure gets worse: both share the same empty
component, release-please logs `Multiple paths for : aws-auth,
private-git-dep`, collapses them, and only one release surfaces. The field is
read per package (`extractReleaserConfig` in `src/manifest.ts`) but is absent
from the published JSON schema, so an editor will not flag its omission.

**`initial-version` is required.** Without it the first release is `1.0.0`, not
`0.1.0` — `initialReleaseVersion()` in release-please returns `1.0.0`, and
`bump-minor-pre-major` does not apply to a first release because it is a
versioning-strategy option and no prior version exists to bump.

Package keys must be **bare directory names** (`aws-auth`, not
`packages/aws-auth`). The release workflow derives each moving major tag from
the tag release-please actually created, so a nested path would still work — but
bare names keep the tag, the component and the directory identical, which is
what the ADR describes.

## Never add a `"."` root package

release-please hands a root package **every** commit in the repository,
bypassing the per-directory split. Adding one would release every action on
every commit, including README-only changes.

If you are here because a README change did not trigger a release: that is
correct and deliberate. A commit touching no package directory releases nothing.

## Version bumps under 0.x

Both pre-1.0 flags are set, so while an action is on `0.x`:

| Commit | Bump | Example |
|---|---|---|
| `feat!:` or `BREAKING CHANGE:` | minor | `0.1.0` → `0.2.0` |
| `feat:` | patch | `0.1.0` → `0.1.1` |
| `fix:` | patch | `0.1.0` → `0.1.1` |

`bump-minor-pre-major` handles the first row; `bump-patch-for-minor-pre-major`
handles the second. Both are needed — with only the first, `feat:` and `feat!:`
produce the same bump and stop being distinguishable.

## What happens on merge to main

1. release-please reads conventional commits since the last release and opens or
   updates a single combined release PR (`separate-pull-requests: false`).
2. Merging that PR tags each released component (`aws-auth/v0.2.1`), writes
   changelogs, and creates GitHub releases.
3. The `move-major-tags` job then force-moves each moving major tag
   (`aws-auth/v0`) to the released commit.

Step 3 is separate because **release-please does not move major tags.** It
creates the exact version tag and stops.

## The release workflow references nothing from this repository

`release.yml` must never use an action from this repo, via `$/` or otherwise.
A broken action would otherwise block the release that fixes it. Ordinary CI
does self-reference — that is the dogfooding signal — but the release path stays
clear.

## Moving tags freeze at a major boundary

When an action reaches `1.0.0`, `aws-auth/v0` stops moving and stays at the last
`0.x` release. There is no LTS: consumers pinned to `v0` keep working but receive
nothing further, including security fixes. The workflow emits a warning on the
first release of a new major.

## Never create a bare tag named after an action

Git cannot hold both a tag `aws-auth` and tags under `aws-auth/`. Creating
`aws-auth` would make every future `aws-auth/vN` tag impossible. CI enforces
this.

## Pinned SHAs, and why the Dependabot glob matters

Every third-party action is pinned to a full SHA with a `# vX.Y.Z` comment. The
comment is not decoration: without it neither Dependabot nor Renovate can tell
what the hash refers to, and updates stop.

Dependabot's default configuration scans `.github/workflows` and a root
`action.yml` only. A pin *inside* a composite action would never be updated
([dependabot-core#6704](https://github.com/dependabot/dependabot-core/issues/6704),
open and unstarted) — which would have silently frozen the
`aws-actions/configure-aws-credentials` pin inside `aws-auth`.

The `directories: ["/", "**/*"]` glob in `.github/dependabot.yml` covers it.
GitHub documents only the single `*` wildcard for this ecosystem and gives no
nested example, so this was **verified empirically** rather than assumed: a
throwaway `probe/action.yml` carrying a deliberately outdated pin was picked up
and bumped in [#10](https://github.com/rockindahizzy/actions/pull/10). The probe
was deleted once it had answered.

Two things worth knowing from that run:

- Dependabot ran on **config detection**, not on the weekly schedule. `interval:
  weekly` governs recurring checks, not the first one.
- Grouping collapsed three directories into a single PR, as intended.

Never automerge these. A dependency update that merges itself is a direct path
into a repository whose actions hold OIDC access to AWS — and the first real
update was a two-major jump, exactly the kind that needs a human read.
