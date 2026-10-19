---
summary: Publishing before versioning and waiting for the version to appear on npm was tried in publish.yml (#17) and reverted within the hour (#18), because neither incident behind it lost a release; publish.yml is back to ebcae1e, which is also what the consumers' `v1` tag points to.
---

# Publish before versioning, and the post-publish npm wait: tried and reverted

## Current state

`.github/workflows/publish.yml` and `README.md` are as at `ebcae1e` (the merge of #16).
The `v1` tag, which every consumer pins (`publish.yml@v1`), points to `ebcae1e`; it was never
moved onto #17, so no repo ever ran the change. In the `Publish Release` job:

1. `Create release Pull Request` (`changesets/action@v1`, no `publish:` input) opens or updates
   the version PR and outputs `hasChangesets`.
2. `Publish` runs only when `hasChangesets != 'true'`: it publishes the version in main's
   `package.json` (OIDC publish or `npm stage publish`), idempotently, skipping a version the
   registry already has or that is staged.
3. `Merge the version PR` runs when `hasChangesets == 'true'`.

A green run does not wait for the registry to serve the version.

## What was tried (#17)

- `Publish` moved before changesets and ran on every push to main with no `if:`, so a run that
  was skipped or cancelled could not lose a version: the next push would publish it.
- After `npm publish`, poll `npm view <name>@<version> --prefer-online` every 15 s for up to
  20 minutes and fail with `::error title=Published version missing::` if it never appears;
  after `npm stage publish`, fail unless `npm stage list <name>` shows the version.
- A README paragraph on what a green Publish run guarantees.

It came from two reports:

- **`@_linked/core` 2.35.1 never reached npm.** #349's changeset reached core's `main` after
  release PR #351 was opened and before it merged (#349's own run was cancelled by the
  concurrency group). The merge run of #351 therefore still saw a pending changeset,
  `hasChangesets` was `true`, `Publish` was skipped, and changesets went on to 2.36.0.
- **`@_linked/server-utils` 1.13.0 looked lost.** Its run logged `+ @_linked/server-utils@1.13.0`
  and was green; the registry served it about 7 minutes later with the logged shasum.

## Why it was reverted (#18)

Neither problem lost a change:

- server-utils 1.13.0 was published by its run; npm taking a few minutes to serve it is normal.
- core 2.35.1 was skipped, but its changes shipped minutes later in 2.36.0, which is `latest`.
  Publishing 2.35.1 afterwards would only add a version below `latest` that nothing resolves to.

A publish job that waits up to 20 minutes on the registry, and a publish step on every push,
add CI time and moving parts without saving a release. Reverted with `git revert ffe6458`;
`git diff ebcae1e HEAD` is empty.

## Known limits (accepted)

- A changeset that lands on `main` while a version PR is open makes that version PR's merge
  skip publishing; the version is folded into the next one.
- A green Publish run means `npm publish` succeeded, not that the registry already serves the
  version; allow a few minutes before treating a version as missing.

## Follow-ups

None. `@_linked/core@2.35.1` stays unpublished on purpose.

## PRs

- https://github.com/linked-fw/.github/pull/17
- https://github.com/linked-fw/.github/pull/18
