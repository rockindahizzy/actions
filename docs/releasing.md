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
    "initial-version": "0.1.0"
  }
}
```

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
