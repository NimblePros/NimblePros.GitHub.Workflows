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
  automatically on every published `v1.x.y` release.
- Tag protection (rulesets on `v*` tags) is deferred: the moving tag has to be force-updated
  by the release workflow, which needs a bypass for GitHub Actions. Noted as follow-up.
- The skill's default verify command (`dotnet build/test/format`) does not apply: this repo
  has no .NET code. Verification is `actionlint` on all workflows instead.

## Phase 1: Protect main (repo settings, done via `gh api`)

- [x] Confirm `main` has no classic protection and no rulesets.
- [ ] Create the `Protect main` ruleset (PR + 1 approval, no force push, no deletion,
      admin bypass via PR only).
- [ ] Confirm the ruleset is active with `gh api repos/{owner}/{repo}/rulesets`.

## Phase 2: Release automation

- [ ] Add `.github/workflows/release.yml`, triggered on `release: published`, that
      validates the tag is `vMAJOR.MINOR.PATCH` and force-moves the `vMAJOR` tag to the
      release commit. Skip prereleases. Least-privilege `contents: write` only.
- [ ] Make sure the release workflow is **not** `workflow_call`-able (it's for this repo
      only).

## Phase 3: Documentation

- [ ] Update every README example from `@main` to `@v1`.
- [ ] Add a "Versioning" section to the README: which ref to use (`@v1`, `@v1.2.3`, or a
      commit SHA), what counts as a breaking change, and how Dependabot keeps `@v1` refs
      current.
- [ ] Add a "Releasing" section for maintainers: how to cut a release and what the release
      workflow does.

## Phase 4: Automated validation

No unit/integration/e2e test projects exist in this repo (YAML only).

- [ ] Run `actionlint` over `.github/workflows/*.yml`; the new `release.yml` must be clean.
- [ ] Test the tag-validation shell logic locally with valid (`v1.0.0`, `v2.10.3`) and
      invalid (`v1`, `1.0.0`, `v1.0`, `v1.0.0-rc.1`) tags.

## Phase 5: Independent review

- [ ] Separate agent (fresh context) reviews the full diff and the ruleset against the
      issue: correctness, missed requirements, security (token scope, injection of the tag
      name into shell), regressions.
- [ ] Address all significant findings, or record why not below.

## Phase 6: Final verification

- [ ] `actionlint` clean on all workflows after the last change.
- [ ] `grep @main README.md` returns no caller examples.

## Follow-up (out of scope)

- After merge: publish `v1.0.0` (`gh release create v1.0.0 --generate-notes`), which moves
  `v1` via the new workflow.
- Tag ruleset for `v*` with GitHub Actions as bypass actor.
- Add CI to this repo (actionlint on PRs) and make it a required status check.

## Review notes

_(filled in after Phase 5)_
