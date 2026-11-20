# NimblePros' Common Workflows

This repo holds common GitHub Actions Workflows that we use in the repositories we maintain.

This makes use of [GitHub's Reusable Workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows).

## Versioning

These workflows run in your repository with your token, so reference a release, not `main`. `main` can change at any time.

| Reference | What you get |
|---|---|
| `@v1` (recommended) | The latest `v1.x.y` release. Moves forward on every non-breaking release. |
| `@v1.2.3` | Exactly that release. Releases are immutable, so it can't change. |
| `@<full commit SHA>` | Exactly that commit. The strictest option, if your policy requires pinning to a SHA. |

Releases follow [Semantic Versioning](https://semver.org/). A new **major** version (`v2`) means a breaking change for callers, such as a removed or renamed input or secret, a new required input, or a change to the permissions the caller must grant. Read the [release notes](https://github.com/NimblePros/NimblePros.GitHub.Workflows/releases) before you move to a new major version.

Dependabot updates a reference when a newer tag of the same form exists: `@v1` gets a pull request when `v2` ships (a breaking change, so read the notes), while `@v1.2.3` gets one for every release. A SHA pin is updated if you keep the version in a trailing comment (`@<sha> # v1.2.3`).

To have Dependabot keep your references up to date, enable the `github-actions` ecosystem in your repository's `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Available Workflows

In this section, we'll mention workflows and any details required for using them.

- [Comment on Pull Request](#comment-on-pull-request)
- [dotnet format](#dotnet-format)
- [Publish to NuGet](#publish-to-nuget)
- [.NET Test Coverage](#net-test-coverage)

---

### Comment On Pull Request

- [Comment on Pull Request workflow](https://github.com/NimblePros/NimblePros.GitHub.Workflows/blob/main/.github/workflows/comment-on-pr.yml)

This workflow posts a code coverage comment on a PR, as seen in [this PR for eShopOnNServiceBus](https://github.com/NimblePros/eShopOnNServiceBus/pull/7).

How it works:

- Call it from a workflow triggered by `workflow_run` on your build workflow. That runs in the base repository with a write token, so fork PRs get comments without giving the PR build write access. It only comments when the triggering build ran for a `pull_request` event and wasn't cancelled.
- It downloads artifacts from the exact build run that triggered the caller (`github.event.workflow_run.id`), so it never posts coverage from a different run or PR.
- The PR number comes from the PR build, so a fork PR controls it. The workflow only accepts it if it's numeric and that PR's head commit matches the commit the build ran on. On a mismatch (for example, a newer push landed first) it skips the comment instead of failing.
- The calling workflow must grant `actions: read` and `pull-requests: write`.

Inputs:

- **code-coverage-artifact-name** (required): Name of the artifact your build uploads with the coverage markdown.
- **code-coverage-artifact-path** (required): Path of the markdown file to post, after the artifact is downloaded.
- **pr-number** (optional): PR number to comment on. Leave it out to have the workflow read the number from an artifact instead.
- **pr-number-artifact-name** (optional, default `pr-number`): Artifact holding the PR number, used when `pr-number` is empty.
- **pr-number-file** (optional, default `pr-number.txt`): File in that artifact containing the PR number.

In your build workflow, upload the coverage markdown and the PR number:

```yaml
    - name: Save the PR number
      if: github.event_name == 'pull_request'
      run: echo ${{ github.event.number }} > pr-number.txt

    - name: Upload the PR number
      if: github.event_name == 'pull_request' && (success() || failure())
      uses: actions/upload-artifact@v7
      with:
        name: pr-number
        path: pr-number.txt

    - name: Upload code coverage results artifact
      if: success() || failure()
      uses: actions/upload-artifact@v7
      with:
        name: code-coverage-results
        path: code-coverage-results.md
```

Then add a caller workflow:

```yaml
name: Comment on the Pull Request

on:
  workflow_run:
    workflows: ["YOUR BUILD WORKFLOW NAME"]
    types:
      - completed

permissions:
  actions: read
  pull-requests: write

jobs:
  add-code-coverage-to-pr:
    uses: NimblePros/NimblePros.GitHub.Workflows/.github/workflows/comment-on-pr.yml@v1
    with:
      code-coverage-artifact-name: code-coverage-results
      code-coverage-artifact-path: code-coverage-results.md
```

---

### dotnet format

- [dotnet format workflow](https://github.com/NimblePros/NimblePros.GitHub.Workflows/blob/main/.github/workflows/dotnet-format.yml)

Runs `dotnet format` against a solution or project. It can either check formatting or apply formatting fixes back to a pull request branch.

```yaml
jobs:
  dotnet-format:
    uses: NimblePros/NimblePros.GitHub.Workflows/.github/workflows/dotnet-format.yml@v1
    with:
      solution-or-project: ./YourSolution.sln
```

---

### Publish to NuGet

- [Publish to NuGet workflow](https://github.com/NimblePros/NimblePros.GitHub.Workflows/blob/main/.github/workflows/dotnet-publish.yml)

Builds, packs, and publishes a .NET project to NuGet.

```yaml
jobs:
  publish:
    uses: NimblePros/NimblePros.GitHub.Workflows/.github/workflows/dotnet-publish.yml@v1
    with:
      project-file: ./src/YourProject/YourProject.csproj
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
```

---

### .NET Test Coverage

- [.NET Test Coverage workflow](https://github.com/NimblePros/NimblePros.GitHub.Workflows/blob/main/.github/workflows/dotnet-test-coverage.yml)

Runs restore, build, test, and coverage reporting for a .NET solution or project.

```yaml
jobs:
  test-coverage:
    uses: NimblePros/NimblePros.GitHub.Workflows/.github/workflows/dotnet-test-coverage.yml@v1
    with:
      solution-path: ./YourSolution.sln
```

---

## Releasing

For maintainers. The repository settings that keep callers safe:

- **Protect main** ruleset: changes reach `main` only through a pull request with one approving review. No force pushes or deletion. Admins can bypass the review, but still only through a pull request.
- **Protect version tags** ruleset: only repository admins and deploy keys can create, move or delete `v*` tags. So only admins can create the tag for a new release. Any write deploy key can bypass this ruleset, so don't add other write deploy keys to this repository.
- **Immutable releases** are on, so a published `vX.Y.Z` tag and its assets can't be changed.
- The `release` environment holds `RELEASE_DEPLOY_KEY`, the private half of the write deploy key the workflow uses to move the major tag. The environment only accepts runs on `v*.*.*` tags, so other workflows can't read it. To rotate the key, add a new write deploy key, replace the secret, then delete the old key.
- The default `GITHUB_TOKEN` is read-only, and GitHub Actions can't approve pull requests.

To publish a release:

1. Merge the changes to `main`.
2. As a repository admin, publish a release whose tag is `vMAJOR.MINOR.PATCH` on a commit that is already on `main`. This is a security requirement, not just a convention: the release run uses `release.yml` from the tagged commit and gets the deploy key, so tagging an unreviewed commit exposes the key. Never create a release named after a major tag such as `v1`: immutable releases would lock that tag so it could no longer move. Bump the major version for any breaking change (see [Versioning](#versioning)). A suffix such as `-rc.1` is only for prereleases.

   ```bash
   gh release create v1.2.0 --target main --generate-notes
   ```

3. The [Move Major Version Tag workflow](https://github.com/NimblePros/NimblePros.GitHub.Workflows/blob/main/.github/workflows/release.yml) then points the major tag (`v1`) at the release commit, so `@v1` callers get it. It runs when a full release is published, or when a prerelease is promoted to a full release. It leaves `v1` alone when:

   - the release is a prerelease or draft,
   - the release is older than the newest `v1.x.y` on `main` (for example a `v1.2.1` hotfix published after `v1.3.0`), so `v1` never moves backwards.

   It fails without moving anything when the tag isn't `vMAJOR.MINOR.PATCH`, or the release commit isn't on `main`.
