# NimblePros' Common Workflows

This repo holds common GitHub Actions Workflows that we use in the repositories we maintain.

This makes use of [GitHub's Reusable Workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows).

## Available Workflows

In this section, we'll mention workflows and any details required for using them.

- [Comment on Pull Request](#comment-on-pull-request)

---

### Comment On Pull Request

- [Comment on Pull Request workflow](https://github.com/NimblePros/NimblePros.GitHub.Workflows/blob/main/.github/workflows/comment-on-pr.yml)

This Pull Request will publish a Code Coverage comment to a PR, as seen in [this PR for eShopOnNServiceBus](https://github.com/NimblePros/eShopOnNServiceBus/pull/7).

Some things to note:

- This only comments on a `pull_request` event.
- This requires 3 parameters:
  - **pr-number**: PR Number to comment on
  - **code-coverage-artifact-name**: Used for downloading hte code-coverage artifact created in a build pipeline
  - **code-coverage-artifact-path**: Used for the commenter plugin

  To use this in your own repo, add this job to your workflow:

```yaml
  add-code-coverage-to-pr:
    uses: NimblePros/NimblePros.GitHub.Workflows/.github/workflows/comment-on-pr.yml@main
    with:
      pr-number: YOUR_PR_NUMBER
      code-coverage-artifact-name: YOUR_CODE_COVERAGE_ARTIFACT_NAME
      code-coverage-artifact-path: YOUR_CODE_COVERAGE_ARTIFACT_PATH
```

You can see this in action with [eShopOnNServiceBus](https://github.com/NimblePros/eShopOnNServiceBus/blob/2012fe78bfbb28fca96609d25176d4cbe0e209ad/.github/workflows/dotnetcore.yml#L57-L63).