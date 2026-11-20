# Issue #6: Protect main and tag releases so callers can reference reusable workflows safely

Issue: https://github.com/NimblePros/NimblePros.GitHub.Workflows/issues/6

## Goal

Callers reference these reusable workflows at `@main`, and `main` is unprotected. Protect
`main`, publish versioned releases with a moving major tag (`v1`), and point the README
examples at `@v1`.

## Assumptions

- Branch protection is applied as a repository **ruleset** on the default branch (the
  newer replacement for classic branch protection): require a pull request with 1
  approval, dismiss stale approvals on push, resolve review threads, block force pushes and
  deletion. Repository admins can bypass **only through a pull request** (so an admin can
  merge in an emergency without a second reviewer, but can't push directly).
- This repo has no required status checks yet (it has no CI of its own), so the ruleset
  doesn't require any.
- The first release (`v1.0.0` and the moving `v1`) is cut by a maintainer **after** this PR
  merges, so the tagged README already shows `@v1`. The new release workflow moves `v1`
  automatically on every full `v1.x.y` release.
- The skill's default verify command (`dotnet build/test/format`) does not apply: this repo
  has no .NET code. Verification is `actionlint` on all workflows, plus local tests of the
  tag script.

## Phase 1: Protect main (repo settings, done via `gh api`)

- [x] Confirm `main` has no classic protection and no rulesets.
- [x] Create the `Protect main` ruleset (PR + 1 approval, no force push, no deletion,
      admin bypass via PR only). Ruleset id 27695121.
- [x] Confirm the rules apply to `main` (`gh api repos/{owner}/{repo}/rules/branches/main`
      returns deletion, non_fast_forward, pull_request).

## Phase 2: Release automation

- [x] Add `.github/workflows/release.yml` that validates the tag is
      `vMAJOR.MINOR.PATCH` and force-moves the `vMAJOR` tag to the release commit. Skip
      prereleases.
- [x] The release workflow is **not** `workflow_call`-able (it's for this repo only).

## Phase 3: Documentation

- [x] Update every README example from `@main` to `@v1` (and the `apicompat.yml` header
      comment).
- [x] Add a "Versioning" section to the README: which ref to use, what counts as a
      breaking change, how Dependabot keeps refs current.
- [x] Add a "Releasing" section for maintainers.

## Phase 4: Automated validation

No unit/integration/e2e test projects exist in this repo (YAML only).

- [x] `actionlint` over `.github/workflows/*.yml` is clean.
- [x] Tag script tested locally against a bare origin: valid tags move the major tag
      (lightweight and annotated, new major creates `v2` without touching `v1`); invalid
      tags (`v1`, `1.0.0`, `v1.0`, `v1.0.0-rc.1`, `v1.0.0;id`) fail; a missing tag fails;
      a release commit not on `main` fails; an older release (`v1.2.1` after `v1.3.0`)
      leaves the major tag alone; an unmerged stray tag doesn't block a legitimate move.

## Phase 5: Independent review

- [x] Separate agent (fresh context) reviewed the full diff and the ruleset.
- [x] Address all significant findings (see Review notes).

## Phase 6: Final verification

- [x] `actionlint` clean on all workflows after the last change.
- [x] No caller example in the README or workflows still references `@main`.

## Follow-up (out of scope)

- After merge: an admin publishes `v1.0.0` (`gh release create v1.0.0 --target main
  --generate-notes`), which moves `v1` via the new workflow. That first run is the
  end-to-end check of the deploy key, environment and tag ruleset.
- Add CI to this repo (actionlint on PRs) and make it a required status check on `main`.
- Optional hardening from the second review: a CODEOWNERS entry for
  `.github/workflows/release.yml` with `require_code_owner_review` in the main ruleset, and
  a required reviewer on the `release` environment.

## Review notes

Findings from the independent review and what was done:

1. **Significant: `v1` could point at unreviewed code.** A release targeting an unmerged
   branch would move `v1`, and anyone with write access could force-push `v1` directly.
   - Workflow now refuses a release commit that isn't on the default branch.
   - Added the **Protect version tags** ruleset (id 27695972) on `refs/tags/v*`: blocks
     creation, update, deletion and force-push, with admins and deploy keys as bypass.
     GitHub rejected the GitHub Actions app as a bypass actor ("must be part of the ruleset
     source or owner organization"), so the workflow pushes with a **write deploy key**
     instead. Its private key is the `RELEASE_DEPLOY_KEY` secret in a new `release`
     environment whose deployment policy only allows `v*.*.*` tags. Only admins can create
     those, so a branch workflow can't read the key.
   - Turned on **immutable releases**, so `@v1.2.3` really can't change.
2. **Significant: an older release moved `v1` backwards.** The workflow now only moves the
   major tag when the release is the newest `vN.x.y` merged into the default branch.
3. **Minor: promoting a prerelease didn't move `v1`.** Trigger changed from `published` to
   `released`, which fires for a direct full release and for a promoted prerelease.
4. **Minor: branch behind `main`, `checkout@v4`.** Rebased; now `actions/checkout@v7` like
   the other workflows.
5. **Minor: `apicompat.yml` header still showed `@main`.** Changed to `@v1`. A README section
   for apicompat is left out: it isn't part of this issue.
6. **Minor: Actions token could write and approve PRs.** Repo default token set to
   read-only and "Allow GitHub Actions to approve pull requests" turned off.
   `release.yml` now only needs `contents: read`.
7. **Minor: workflow-file push restriction.** Moot: the release commit must be on `main`,
   and the push uses the deploy key.
- Nits: README now explains Dependabot behavior per ref type, the release rules, and the
  repo protections. Merge methods were left at all three: no team preference recorded.

Second review (same reviewer, after the fixes above): nothing significant. README now warns
that tagging an unreviewed commit exposes the deploy key, that any write deploy key can bypass
the tag ruleset, that only admins create release tags (write users can still publish a
release for an existing tag, which is harmless), and not to create a release named `v1`.
CODEOWNERS and an environment reviewer are left as optional follow-up: they add process
beyond what the issue asked for.
