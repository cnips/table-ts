# Releasing @cnips/table

Releases are automatic. Merge to `main` and the version, the tag, the npm
publish and the release notes follow from the commits. **Do not create tags by
hand** (except the one-time bootstrap tag below).

## How a release happens

Every push to `main` runs `.github/workflows/ci.yml`:

1. **test**: `npm ci`, `npm run build` and `npm test`, on Node LTS and Node 24.
2. **release**, only if `test` passed:
   [semantic-release](https://github.com/semantic-release/semantic-release)
   reads the commits since the last tag and decides the next version. It then
   publishes `@cnips/table` to npm, pushes the tag `vX.Y.Z`, and creates a
   [GitHub release](https://github.com/cnips/table-ts/releases) whose notes
   are generated from those commits.
3. The new version is requested from the npm registry so it is visible
   immediately.

## How the version is decided

semantic-release takes the largest bump any commit since the last tag asks
for ([Conventional Commits](https://www.conventionalcommits.org/)):

| Commit | Example | Release |
| --- | --- | --- |
| `feat` | `feat(accessor): add bulkInsert (#12)` | **minor**, 1.2.0 → 1.3.0 |
| `fix`, `perf` | `fix(find): handle empty query (#1)` | **patch**, 1.2.0 → 1.2.1 |
| `!` after the type, or a `BREAKING CHANGE:` footer | `feat(accessor)!: drop insertRow (#20)` | **major**, 1.2.0 → 2.0.0 |
| `docs`, `test`, `ci`, `chore`, `refactor`, `build`, `style` | `docs(readme): fix example` | none |

A push that contains only non-releasing commits produces no release. The job
still passes.

**The commit type is a release decision.** A change a caller can notice is a
`fix` or a `feat`, never a `chore` or a `refactor`. Otherwise it does not ship
until something else triggers a release.

Every commit on `main` counts, including each commit of a merged pull request,
not only its title. If a branch's history does not describe what should be
released, squash-merge it and give it a conventional title.

## Why tags must not be made by hand

npm refuses to republish a version that already exists. A tag that was a
mistake cannot be withdrawn once anyone has fetched it. semantic-release only
ever tags a commit that passed the tests, with the next version in sequence.

## Setup

1. Add a repository secret `NPM_TOKEN` with an npm automation token that can
   publish `@cnips/table`.
2. The job uses the workflow's built-in `GITHUB_TOKEN` with `contents: write`.
   A tag pushed with that token does not trigger further workflows.

Release comments on issues and pull requests, and "released" labels, are
turned off in `.releaserc.json`, so the job needs no permissions beyond
`contents: write`.

### Bootstrap (one-time)

`@cnips/table` is already published through 1.0.3, but this repo has no `v*`
tags yet. Before the first automated release, create a tag that matches the
current npm version so semantic-release continues from there:

```bash
git tag v1.0.3 <commit-that-matches-1.0.3>
git push origin v1.0.3
```

Without that tag, the next `feat` / `fix` on `main` would try `1.0.0` and
fail against npm.

## Verifying a release

```bash
gh release view vX.Y.Z --repo cnips/table-ts
npm view @cnips/table version
npm view @cnips/table versions --json
```

## Pinned versions

The actions are pinned by commit SHA and the npm packages by exact version:
`semantic-release@25.0.9` and `conventional-changelog-conventionalcommits@9.3.1`.
semantic-release's own plugins (including `@semantic-release/npm`) resolve
within the ranges that version declares. Upgrade deliberately, and let the
next release prove it.

The preset and semantic-release have to be upgraded together.
`conventional-changelog-conventionalcommits` 10.x renders only with
`conventional-changelog-writer` 9. semantic-release 25.0.9 ships
`@semantic-release/release-notes-generator` 14, which loads writer 8, so a 10.x
preset fails at "generateNotes" before anything is tagged.

To test an upgrade locally, run a dry run with the real `.releaserc.json`.
Do not override plugins with `--plugins`: that drops the preset options and
tests the default preset instead.
