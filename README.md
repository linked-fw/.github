# linked-fw/.github

The two reusable GitHub Actions workflows that every `@_linked` package repo runs, plus this
org's public profile page (`profile/README.md`).

This README is for maintainers of *these* workflows. What a package repo needs to know about
publishing is on the profile page, and in Create Now's
`docs/architecture/03-packages-and-governance.md`.

| File | Called as |
|---|---|
| `.github/workflows/pr.yml` | `linked-fw/.github/.github/workflows/pr.yml@v1` |
| `.github/workflows/publish.yml` | `linked-fw/.github/.github/workflows/publish.yml@v1` |

Both are `on: workflow_call` only. They never run on this repo's own PRs, which is why this
repo requires no status check.

## Who consumes them

**22 repos in this org**, each carrying a small caller stub for both workflows, every one pinned
to `@v1`:

`auth`, `cli`, `core`, `css`, `dcat`, `dcmi`, `fuseki`, `icons`, `org`, `owl`, `primitives`,
`rdfs`, `react`, `s3`, `schema`, `sentry`, `server`, `server-utils`, `shape-ui`, `sioc`,
`translation`, `xsd`.

Four repos in the org do **not**: `app-template`, `livekit`, `live-sessions` (no caller stubs at
all) and this repo. `linked-cm` keeps its own copy of both workflows at its own `v1` tag — it
does not call these, so a change here does not reach community packages.

## Inputs a caller can pass

### `pr.yml`

| Input | Type | Default | What it does |
|---|---|---|---|
| `node-version` | string | `24.21.0` | Node for build and test. |
| `require-tests` | boolean | `false` | `false` runs `npm run <test-script> --if-present`, so a package with no test script passes. `true` makes a missing script a failure — for anything with real logic. |
| `test-script` | string | `test` | Which npm script the Test step runs. For a package whose `test` chains a slow or container-backed suite, name the unit script here and put the rest behind `run-e2e`. |
| `run-e2e` | boolean | `false` | Additionally runs `npm run test:e2e --if-present`. Opt-in: these start containers. |

No secrets. The caller must grant `pull-requests: write` for the changeset reminder comment — a
reusable workflow cannot widen what its caller holds.

### `publish.yml`

| Input | Type | Default | What it does |
|---|---|---|---|
| `node-version` | string | `24.21.0` | Node for build and publish. |
| `npm-version` | string | `^11.15.0` | npm installed globally before the run. `npm stage` needs ≥ 11.15.0; Node 24.21 already bundles npm 11.19, so this is now a pin rather than a fix — it keeps the npm a release runs on decided here, not by the Node image. |

Required secrets: `NPM_TOKEN` (stage-only fallback; unused when the package has a trusted
publisher), `RELEASE_APP_ID`, `RELEASE_APP_PRIVATE_KEY`. Pass them by name, never
`secrets: inherit`. The caller must grant `contents: write`, `pull-requests: write` and
`id-token: write`.

## Releasing a change: merging is not enough

**Merging a PR into `main` here reaches nobody.** Every consumer is pinned to `@v1`, and `v1` is
an annotated tag that is moved by hand. Until someone moves it, the change sits on `main` doing
nothing — which is a quiet failure, because the PR looks shipped.

Releasing is therefore two acts, and the second one is manual:

```bash
# 1. Merge the PR into main, as normal. Then:
git fetch origin
git checkout main && git pull --ff-only

# 2. Move v1 onto the new tip and force-push the tag.
git tag -fa v1 "$(git rev-parse HEAD)" -m v1
git push --force origin v1
```

Check what you are about to do first:

```bash
git log --oneline "$(git rev-parse v1^{commit})"..origin/main   # what this release contains
```

An empty result means `v1` is already current and there is nothing to release.

### Blast radius

The tag move is **immediate and fleet-wide**. All 22 repos pick up the new workflow on their very
next run. There is no staged rollout, no per-repo opt-in, and no way for one repo to stay behind
short of editing its own stub.

`publish.yml` is the sharper of the two: it sits on the release path, it merges the changesets
version PR itself, and it publishes to npm. A bad tag move there can break or mis-publish a
release for every first-party package at once. `pr.yml` fails more safely — it blocks merges
rather than shipping something wrong — but a broken `Build & Test` still stalls every release,
because the automated version PR can never satisfy a check that does not report.

Prefer to release a `publish.yml` change when you can watch the next release happen.

### Rolling back

Symmetrical — one more force-move, back to the previous commit:

```bash
git tag -fa v1 <previous-sha> -m v1
git push --force origin v1
```

Consumers pick the rollback up on their next run, with no change in any consumer repo. Find the
previous sha in this repo's history, or in the `Setup job` log of a consumer run from before the
move (see below).

### Verifying it took effect

The tag itself — note that `refs/tags/v1` names an *annotated tag object*, not the commit, so
resolving it takes two hops:

```bash
git ls-remote --tags origin v1
gh api repos/linked-fw/.github/git/refs/tags/v1 --jq '.object.sha'   # tag object
gh api repos/linked-fw/.github/git/tags/<tag-object-sha> --jq '.object.sha'   # the commit
```

Then look at a consumer. Open the next `PR` or `Publish` run in, say, `linked-fw/core` and read
the **`Set up job`** step. It logs the version it resolved:

```
Uses: linked-fw/.github/.github/workflows/publish.yml@refs/tags/v1 (fd6786b05e3229958b8721def75c13a94350a95f)
```

That sha is the tag object's, and it changes every time the tag is force-moved — so comparing it
against the sha in a run from before the move is the proof the move landed. The `@v1` text in the
caller stub never changes and tells you nothing. Then confirm in the same run's log that the step
you actually changed behaves as expected.

A run already in flight keeps the version it resolved at start.

## Recommendation (not current practice)

A moving major tag is the standard Actions convention — `actions/checkout@v4` works exactly this
way — so the convention is fine. The gaps are that it is **undocumented** (this file is the fix)
and **unautomated**: today the only record that a release happened is the tag's own history, and
there are no releases and no `v1.x.y` tags to roll back *to* by name.

What I would do, in order of value:

1. **Cut an immutable `v1.N.M` tag alongside each `v1` move, and a GitHub Release for it.** Cheap,
   and it gives rollback a name and the fleet a changelog. `v1` keeps moving and consumers keep
   their `@v1` pin.
2. **Automate the move**: a workflow on push to `main` that re-points `v1` and cuts the release,
   so "merged" and "released" stop being two different states nobody can tell apart. Add it after
   (1), not before — automating an unrecorded move makes it easier to do and no easier to undo.
3. **Do not pin consumers to a sha.** It would hand back the 44-file edit across 22 repos that
   consolidating these workflows removed, and it buys safety only if someone actually bumps the
   shas, which nobody will.

Until (2) exists, the manual step above is the release process, and skipping it is the mistake
this README is written to prevent.
