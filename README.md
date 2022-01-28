![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/levibostian/action-semantic-release-justwork?label=latest%20stable%20release)
![GitHub release (latest SemVer including pre-releases)](https://img.shields.io/github/v/release/levibostian/action-semantic-release-justwork?include_prereleases&label=latest%20pre-release%20version)

# action-semantic-release-justwork

GitHub Action to perform setup steps for semantic-release tool to avoid common pitfalls and errors. 

# Getting started 

This Action does not *run* semantic-release for you as it does not know the version, plugins, etc that you like to use. Simply put, run this Action before you run your semantic-release step in your workflow. 

```yml
name: Deploy action 
on:
  push:
    branches: [main, beta, alpha]

jobs:
  deploy:
    name: Deploy 
    runs-on: ubuntu-latest
    steps:
      - name: semantic-release setup
        uses: levibostian/action-semantic-release-justwork@v1
      # You can use whatever method that you wish to run semantic-release. 
      # This action is my current go-to to run it. 
      - name: Deploy via semantic release 
        uses: cycjimmy/semantic-release-action@v2
        with: 
          # version numbers below can be in many forms: M, M.m, M.m.p
          semantic_version: 18
          extra_plugins: |
            @semantic-release/git@10
            @semantic-release/changelog@6
        env:
          GITHUB_TOKEN: ${{ secrets.BOT_PUSH_TOKEN }}
```

# How does this Action work? 

This action performs a series of steps to address a series of pitfalls/errors that you may encounter when running semantic-release. 

### Problem 1: Git operations performed by semantic-release do not trigger other GitHub Action Workflows in your project. 

For example, if you have a workflow that is meant to execute *after a git tag is created* by semantic-release then you need to perform some special setup to make this happen. 

Let me explain. You are probably running `actions/checkout` before you run semantic-release, right? When the Action `actions/checkout` runs, it uses the [`GITHUB_TOKEN` secret](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#about-the-github_token-secret) to authenticate the Workflow. This special token [purposely does not trigger other GitHub Workflows](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#using-the-github_token-in-a-workflow) to avoid a potential infinite loop of running Actions. 
> When you use the repository's GITHUB_TOKEN to perform tasks, events triggered by the GITHUB_TOKEN will not create a new workflow run.

...the solution is to have `git` authenticate using a [GitHub personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) instead of `GITHUB_TOKEN`. semantic-release is setup to do this already by you override `GITHUB_TOKEN` or `GH_TOKEN` with a personal access token when you run it. However, by default, when `actions/checkout` runs, it will *cache the `GITHUB_TOKEN`* which means that when semantic-release runs, `git` will ignore the `GITHUB_TOKEN` that you overrode for semantic-release. 

The solution to this problem is to [disable credentials caching for `actions/checkout`](https://github.com/semantic-release/semantic-release/blob/2c30e268f9484adeb2b9d0bdf52c1cd909779d64/docs/recipes/ci-configurations/github-actions.md#pushing-packagejson-changes-to-a-master-branch) so semantic-release can use it's own credentials. 

### Problem 2: Your repository continuously updates a `latest`, `snapshot`, `v1`, etc tag. 

semantic-release will encounter an error at startup after you...
1. Release a new git tag with semantic-release. For example, `latest` or `v1`. 
2. You overwrite that existing tag in the future. So, push `latest` or `v1` with new commits. 
3. Run semantic-release for a new deployment. It will fail because when it tries to fetch tags, it will fail and recieve a git error `! [rejected]  v1  -> v1  (would clobber existing tag)`. 

GitHub Actions are a perfect example of this flow where even GitHub [recommends that you continuously update a git tag](https://docs.github.com/en/actions/creating-actions/releasing-and-maintaining-actions) stating:
> We recommend creating releases using semantically versioned tags – for example, v1.1.3 – and keeping major (v1) and minor (v1.1) tags current to the latest appropriate commit.

There is [a good discussion](https://github.com/semantic-release/git/issues/136) about why semantic-release does not recommend that you continuously update tags. Good point, but there are some use cases where it provides value and can be done. 

The solution to this problem is to force fetch all tags on the remote git repository before running semantic-release. 

# Development 

This action is a [composite GitHub Action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) mostly relying on bash scripts to run commands. This means there is nothing for you to install to create a development environment on your computer. However, this also means that testing the action is more difficult. To test this action, we rely on running the action on GitHub Actions.

All changes made to the code require making a pull request into `develop` branch with the title conforming to the [conventional commit format](https://www.conventionalcommits.org/).

# Deployment

Tags/releases are made automatically using [semantic-release](https://github.com/semantic-release/semantic-release) as long as our git commit messages are written in the [conventional commit format](https://www.conventionalcommits.org/). Just `git rebase ...` or `git merge` commits from `develop` into `alpha`, `beta`, or `main` to make a new deployment of the action. 


