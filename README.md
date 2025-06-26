# NimblePros' Common Workflows

This repo holds common GitHub Actions Workflows that we use in the repositories we maintain.

This makes use of [GitHub's Reusable Workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows).

## Available Workflows

In this section, we'll mention workflows and any details required for using them.

- [Comment on Pull Request](#comment-on-pull-request)

### Comment On Pull Request

- [Comment on Pull Request workflow](https://github.com/NimblePros/NimblePros.GitHub.Workflows/blob/main/.github/workflows/comment-on-pr.yml)

This Pull Request will publish a Code Coverage comment to a PR, as seen in [this PR for eShopOnNServiceBus](https://github.com/NimblePros/eShopOnNServiceBus/pull/7).

Some things to note:

- This only comments on a `pull_request` event.
- This is highly dependent on the presence of two artifacts: `pr-number` and `code-coverage-results`.

This is the code in our build pipeline for generating the `pr-number` artifact:

```yaml
    - name: Save the PR number in an artifact
      if: github.event_name == 'pull_request' && (success() || failure()) 
      shell: bash
      env:
        PR_NUMBER: ${{ github.event.number }}
      run: echo $PR_NUMBER > pr-number.txt
  
    - name: Upload the PR number
      uses: actions/upload-artifact@v4
      if: github.event_name == 'pull_request' &&  (success() || failure())
      with:
        name: pr-number
        path: ./pr-number.txt
        retention-days: 1
 ```

This is the code in our build pipeline for generating the `code-coverage-results` artifact:

    ```yaml
          # See https://josh-ops.com/posts/github-code-coverage/
      # Add coverlet.collector nuget package to test project - 'dotnet add <TestProject.cspoj> package coverlet
    - name: Test with dotnet
      run: dotnet test ./YOUR_SOLUTION_NAME.sln --no-restore --verbosity normal --collect:"XPlat Code Coverage" --logger trx --results-directory coverage
      
    - name: Copy Coverage To Predictable Location
      run: find coverage -type f -name coverage.cobertura.xml -exec cp -p {} coverage/coverage.cobertura.xml \;
      
    - name: Code Coverage Summary Report
      uses: irongut/CodeCoverageSummary@v1.3.0
      with:
        filename: coverage/coverage.cobertura.xml
        badge: true
        format: 'markdown'
        output: 'both'

    - name: Upload code coverage results artifact
      uses: actions/upload-artifact@v4
      if: success() || failure()
      with:
        name: code-coverage-results
        path: code-coverage-results.md
        retention-days: 1
    ```