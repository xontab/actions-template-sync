# actions-template-sync
<!-- ALL-CONTRIBUTORS-BADGE:START - Do not remove or modify this section -->
[![All Contributors](https://img.shields.io/badge/all_contributors-22-orange.svg?style=flat-square)](#contributors-)
<!-- ALL-CONTRIBUTORS-BADGE:END -->

 [![actions-template-sync](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/actions_template_sync.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/actions_template_sync.yml)

[![Lint](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/lint.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/lint.yml)

[![shellcheck](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/shellcheck.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/shellcheck.yml)

[![test](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/test.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/test.yml)

[![test-hooks](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/test_hooks.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/test_hooks.yml)

[![test-ssh](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/test_ssh.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/test_ssh.yml)

[![test-ssh-gitlab](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/test_ssh_gitlab.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/test_ssh_gitlab.yml)

[![push-docker](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/push_docker.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/push_docker.yml)

[![gh-pages-mk-docs](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/gh_pages_mk_docs.yml/badge.svg)](https://github.com/AndreasAugustin/actions-template-sync/actions/workflows/gh_pages_mk_docs.yml)

## abstract

Synchronise git repositories in an automated manner. Different git providers like GitHub (enterprise), GitLab,.. are supported as the source provider.
This can help you e.g. for migration from another git provider to GitHub or if you want to mirror git repositories.

### History

It is possible to create repositories within Github with
[GitHub templates](https://docs.github.com/en/github/creating-cloning-and-archiving-repositories/creating-a-template-repository).
This is a nice approach to have some boilerplate within your repository.
Over time, the template repository will get some code changes.
The problem is that the already created repositories won't know about those changes.
This GitHub action will help you to keep track of the template changes.
The initial author of this repository faced that issue several times and decided to write a GitHub action to face that issue.
Because of the nice community, several feature requests helped to go on with the development of the action. Now several other features are supported.

## Features

```mermaid
flowchart LR
    github_source("fa:fa-github <b>GitHub</b> source repository <b>[private|public]</b>")
    gitlab_source("fa:fa-gitlab <b>GitLab</b> source repository <b>[private|public]</b>")
    any_source("fa:fa-git <b>Any</b> git provider <b>[private|public]</b>")
    github_target{{"fa:fa-github <b>GitHub</b> target repository <b>[private|public]</b>"}}
    github_source --> |"<b>ssh | PAT | github app</b>"| github_target
    gitlab_source --> |"<b>ssh</b>"| github_target
    any_source --> |"<b>ssh</b>"| github_target
```

* Sync other public or private repository (e.g. template repositories) with the current repository
* Ignore files and folders from syncing using a `.templatesyncignore` file
* many configuration options
* different lifecycle hooks are supported. This opens the possibility to inject custom code into the workflow with a yaml definition file.
* different git provider like GitLab, Gittea,.. as source are supported (with ssh).
  See [.github/workflows/test_ssh_gitlab.yml](.github/workflows/test_ssh_gitlab.yml) for an example.
* It is not necessarily needed that source and target repository have the same base history.
  Because of that reason, it is possible to merge 2 totally different repositories with the help of the action.

## Usage

Usage changes depending on whether the template repository is public or private, regardless of the visibility of the current repository.

### Public template repository

Add this configuration to a GitHub action in the current repository:

```yaml
# File: .github/workflows/template-sync.yml

on:
    # cronjob trigger
  schedule:
  - cron:  "0 0 1 * *"
  # manual trigger
  workflow_dispatch:
jobs:
  repo-sync:
    runs-on: ubuntu-latest
    # https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs
    permissions:
      contents: write
      pull-requests: write

    steps:
      # To use this repository's private action, you must check out the repository
      - name: Checkout
        uses: actions/checkout@v4
        # https://github.com/actions/checkout#usage
        # uncomment if you use submodules within the source repository
        # with:
        #   submodules: true

      - name: actions-template-sync
        uses: AndreasAugustin/actions-template-sync@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          source_repo_path: <owner/repo>
          upstream_branch: <target_branch> # defaults to main
          pr_labels: <label1>,<label2>[,...] # optional, no default
```

You will receive a pull request within your repository if there are some changes available in the template.

### Private template repository

If your current repository was created from a private template, there are several possibilities.

#### 1. Using a GitHub app

You can create and use a [GitHub App][github-app] to handle access to the private template repository.
To generate a token for your app you can use a separate action like [tibdex/github-app-token][github-app-token].
You have to set up the checkout step with the generated token as well.

```yaml
jobs:
  repo-sync:
    runs-on: ubuntu-latest

    steps:
     - name: Generate token to read from source repo # see: https://github.com/tibdex/github-app-token
        id: generate_token
        uses: tibdex/github-app-token@v1
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.PRIVATE_KEY }}

      - name: Checkout
        uses: actions/checkout@v4
        with:
          # submodules: true
          token: ${{ steps.generate_token.outputs.token }}

      - name: actions-template-sync
        uses: AndreasAugustin/actions-template-sync@v1
        with:
          github_token: ${{ steps.generate_token.outputs.token }}
          source_repo_path: <owner/repo>
          upstream_branch: <target_branch> # defaults to main
          pr_labels: <label1>,<label2>[,...] # optional, no default
```

#### 2. Using SSH

You have various options to use ssh keys with GitHub.
An example is [deployment keys][deployment-keys]. For our use case, write permissions are not needed.
Within the current repository, where the GitHub action is enabled, add a secret
(e.q. `SOURCE_REPO_SSH_PRIVATE_KEY`) with the content of your private SSH key.
Make sure that the read permissions of that secret fulfill your use case.
Set the optional `source_repo_ssh_private_key` input parameter.
It is also possible to use a different git provider, e.g. GitLab.

```yaml
jobs:
  repo-sync:
    runs-on: ubuntu-latest
    # https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs
    permissions:
      contents: write
      pull-requests: write

    steps:
      # To use this repository's private action, you must check out the repository
      - name: Checkout
        uses: actions/checkout@v4
        with:
          # submodules: true
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: actions-template-sync
        uses: AndreasAugustin/actions-template-sync@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          source_repo_path: ${{ secrets.SOURCE_REPO_PATH }} # <owner/repo>, should be within secrets
          upstream_branch: ${{ secrets.TARGET_BRANCH }} #<target_branch> # defaults to main
          pr_labels: <label1>,<label2>[,...] # optional, no default
          source_repo_ssh_private_key: ${{ secrets.SOURCE_REPO_SSH_PRIVATE_KEY }} # contains the private ssh key of the private repository
```

#### 3. Using a PAT

:warning: when the source repository is private using PATs, also the target repository must be private.
Else it won't work.

[Personal access token][github-pat] is an alternative to using passwords for authentication to GitHub. You can add a kind
of password to your GitHub account. You need to set the scopes.

* `repo` -> all
* `read:org`

![pat-scopes](docs/assets/pat_needed_scopes.png)

Furthermore, you need to set the access within the source repository to allow GitHub actions within the target repository.
As mentioned before (you can see the note in the image) you need to set the target repository to private.
settings -> actions -> general.

![pat-source-repo-access](docs/assets/pat_needed_access_source_repo.png)

example workflow definition

```yml
name: actions-template-sync

on:
  # cronjob trigger At 00:00 on day-of-month 1. https://crontab.guru/every-month
  schedule:
  - cron:  "0 0 1 * *"
  # manual trigger
  workflow_dispatch:

jobs:
  test-implementation-job:

    runs-on: ubuntu-latest

    steps:
      # To use this repository's private action, you must check out the repository
      - name: Checkout
        uses: actions/checkout@v4
        with:
          # submodules: true
          token: ${{ secrets.CUSTOM_GITHUB_PAT }}

      - name: Test action step PAT
        uses: AndreasAugustin/actions-template-sync@v1
        with:
          github_token: ${{ secrets.CUSTOM_GITHUB_PAT }}
          source_repo_path: ${{ secrets.SOURCE_REPO_PATH }} # <owner/repo>, should be within secrets
```

### Configuration parameters

| Variable | Description | Required | `[Default]` |
|----|----|----|----|
| github_token | Token for the repo. Can be passed in using `$\{{ secrets.GITHUB_TOKEN }}` | `true` |  |
| source_repo_path | Repository path of the template | `true` | |
| upstream_branch | The target branch | `false` | `<The_remote_default>` |
| source_repo_ssh_private_key | `[optional]` private ssh key for the source repository. [see](#private-template-repository)| `false` |  |
| pr_branch_name_prefix | `[optional]` the prefix of branches created by this action | `false` | `chore/template_sync`  |
| pr_title | `[optional]` the title of PRs opened by this action. Must be already created. | `false` | `upstream merge template repository`  |
| pr_labels | `[optional]` comma separated list. [pull request labels][pr-labels]. Must be already created. | `false` | |
| pr_reviewers | `[optional]` comma separated list of pull request reviewers. | `false` | |
| pr_commit_msg | `[optional]` commit message in the created pull request | `false` | `chore(template): merge template changes :up:` |
| hostname | `[optional]` the hostname of the repository | `false` | `github.com` |
| is_dry_run | `[optional]` set to `true` if you do not want to push the changes and not want to create a PR |  `false` |   |
| is_allow_hooks | `[optional]` set to `true` if you want to enable lifecycle hooks. Use this with caution! | `false` | `false` |
| is_not_source_github | `[optional]` set to `true` if the source git provider is not GitHub | `false` | `false` |
| git_user_name | `[optional]` set the committer git user.name | `false` | `${GITHUB_ACTOR}` |
| git_user_email | `[optional]` set the committer git user.email | `false` | `github-action@actions-template-sync.noreply.${SOURCE_REPO_HOSTNAME}` |
| git_remote_pull_params |`[optional]` set remote pull parameters | `false` | `--allow-unrelated-histories --squash --strategy=recursive -X theirs` |

### Docker

There are docker images available. Please checkout [How to use docker](docs/DOCKER.md) for details.

### Example

This repo uses this [template][template] and this action from the [marketplace][marketplace].
See the definition [here][self-usage].

If you look for a more detailed guide you can have a look at

* [Dev.to][devto-example]
* [GitHub][github-example]
* :heart: [nice blog post][dotdc-blog]

### Trigger

You can use all [triggers][action-triggers] which are supported for GitHub actions

## Ignore Files

Create a `.templatesyncignore` file. Just like writing a `.gitignore` file, follow the [glob pattern][glob-pattern]
in defining the files and folders that should be excluded from syncing with the template repository.

It can also be stored inside `.github` folder.

_Note: It is not possible to sync also the `.templatesyncignore` itself. Any changes from the template repository will be restored automatically._

**Remark** reading the [gitglossary][git-glossary] (_pathspec_ section) you see a slight difference to the `.gitignore` file
when you like to disable files you need to use `:!`.
E.g. when you like to disable the sync for all files with exceptions, you need to do smth like

```txt
:!newfile-1.txt
*
```

## Lifecycle hooks

Different lifecycle hooks are supported. You need to enable the functionality with the option `is_allow_hooks` and set it to `true`
:warning: use this functionality with caution. You can use one of the available docker images to test it out. **With great power comes great responsibility**.

* [dockerhub andyaugustin/actions-template-sync][dockerhub-repo]
* [github andreasaugustin/actions-template-sync][github-repo]

In addition, you need a configuration file with the name `templatesync.yml` within the root of the target repository.

The following hooks are supported (please check [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for a better understanding of the lifecycles).

* `install` is executed after the container has started and after reading and setting up the environment.
* `prepull` is executed before the code is pulled from the source repository
* `precommit` is executed before the code is commited
* `prepush` is executed before the push is executed, right after the commit
* `prepr` is executed before the PR is done

**Remark** The underlying OS is defined by an Alpine container.
E.q. for the installation phase you need to use commands like `apk add --update --no-cache python3`

Schema and example for the `templatesync.yml`

```yml
hooks:
  install:
    commands:
      - apk add --update --no-cache python3
      - python3 --version
  prepull:
    commands:
      - echo 'hi, we are within the prepull phase'
      - echo 'maybe you want to do adjustments on the local code'
  precommit:
    commands:
      - echo 'hi, we are within the precommit phase'
      - echo 'maybe you want to add further changes before the code is committed'
  prepush:
    commands:
      - echo 'hi, we are within the prepush phase'
      - echo 'maybe you want to add further changes and commits'
  prepr:
    commands:
      - echo 'hi, we are within the prepr phase'
      - echo 'maybe you want to change the code a bit and do another push before creating the pr'
```

## Troubleshooting

* refusing to allow a GitHub App to create or update workflow `.github/workflows/******.yml` without `workflows` permission

  This happens because the template repository is trying to overwrite some files inside `.github/workflows/`.

  Currently `GITHUB_TOKEN` can't be given `workflow` permission.
  You can grant our workflow with `workflows` permission using a PAT following the steps below:

  1. [Create a PAT][github-create-pat] with these repository permissions granted: `contents:write`, `workflows:write`, `metadata:read`.

  2. Copy the generated token and [create a new secret for your target repository][github-create-secret].

  3. Configure the `checkout` action to use the token in secrets like this:

     ```yaml
     # File: .github/workflows/template-sync.yml

     on:
         # cronjob trigger
       schedule:
       - cron:  "0 0 1 * *"
       # manual trigger
       workflow_dispatch:
     jobs:
       repo-sync:
         runs-on: ubuntu-latest
         # https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs
         permissions:
           contents: write
           pull-requests: write

         steps:
           # To use this repository's private action, you must check out the repository
           - name: Checkout
             uses: actions/checkout@v4
             with:
               # submodules: true
               token: ${{ secrets.<secret_name> }}

           - name: actions-template-sync
             uses: AndreasAugustin/actions-template-sync@v1
             with:
               github_token: ${{ secrets.GITHUB_TOKEN }}
               source_repo_path: <owner/repo>
               upstream_branch: <target_branch> # defaults to main
               pr_labels: <label1>,<label2>[,...] # optional, no default
     ```

* pull request create failed: GraphQL: GitHub Actions is not permitted to create or approve pull requests (createPullRequest)

  Open your project `Settings > Actions > General` and select the checkbox `Allow GitHub Actions to create and approve pull requests`
under the `Workflow permissions` section.

## Release Updates

* :warning: starting with version `v1` (`v1.0.0`) the `upstream_branch` variable default is not `main` anymore. It is now set to the remote default branch.
* starting with version v0.5.2-draft the `templateversionrc` file is not needed anymore. You can delete that file from the target repositories.

## Debug

You must create a secret named `ACTIONS_STEP_DEBUG` with the value `true` to see the debug messages set by this command in the log.
For more information, see "[Enabling debug logging.][enabling-debug-logging]"

## Comparison with other tools

There are other great tools available within GitHub. Here you can find a comparison.

| **feature** | **actions-template-sync** |[github-sync][other-repo-sync]| [git-repo-sync][other-git-repo-sync] | [action-template-repository-sync][other-action-template-repository-sync] |
| ----------- | ------------------------- | -------------------------- | ------------------------------------ | --------------------------------- |
| GitHub action | :heavy_check_mark: | :heavy_check_mark: | :x: | :heavy_check_mark: |
| hooks | :heavy_check_mark: | :x: | :x: | :x: |
| available docker image | :heavy_check_mark: | :x: | :x: | :heavy_check_mark: |
| sync between private and public repo | :heavy_check_mark: `PAT,ssh,Github app` | :heavy_check_mark: `PAT,ssh` |:x: local repos | :heavy_check_mark: `PAT` |
| sync between 2 private repos | :heavy_check_mark: `PAT,ssh,Github app` | :heavy_check_mark: `PAT,ssh` | :x: local repos | :heavy_check_mark: `PAT` |
| sync between 2 public repos | :heavy_check_mark: | :heavy_check_mark: | :x: local repos | :heavy_check_mark: |
| two way sync | :x: | :heavy_check_mark: | :x: | :x: |
| Sync from a third-party repo to a Github repo | :heavy_check_mark: | :heavy_check_mark: | :x: local repos | :x: |
| dry run | :heavy_check_mark: | :x: | :x: | :heavy_check_mark:  |
| ignore files | :heavy_check_mark: | :x: | :x: | :heavy_check_mark: |
| creates a PR | :heavy_check_mark: | :heavy_check_mark: | :x: | :heavy_check_mark: |
| remarks | The action is placed within the target repositories | The action is placed within the target repositories | CLI meant for local use | The action will be based within the base repository with a list of dependent repositories |

## DEV

The development environment targets are located in the [Makefile](Makefile)

```bash
make help
```

For some architectural notes please have a look at the [docs](./docs/README.md)

## Contributors ✨

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/AndreasAugustin"><img src="https://avatars0.githubusercontent.com/u/8027933?v=4?s=100" width="100px;" alt="andy Augustin"/><br /><sub><b>andy Augustin</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=AndreasAugustin" title="Documentation">📖</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=AndreasAugustin" title="Code">💻</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/pulls?q=is%3Apr+reviewed-by%3AAndreasAugustin" title="Reviewed Pull Requests">👀</a> <a href="#security-AndreasAugustin" title="Security">🛡️</a> <a href="#ideas-AndreasAugustin" title="Ideas, Planning, & Feedback">🤔</a> <a href="#question-AndreasAugustin" title="Answering Questions">💬</a> <a href="#example-AndreasAugustin" title="Examples">💡</a> <a href="#content-AndreasAugustin" title="Content">🖋</a> <a href="#blog-AndreasAugustin" title="Blogposts">📝</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://www.iit.it/people/ugo-pattacini"><img src="https://avatars.githubusercontent.com/u/3738070?v=4?s=100" width="100px;" alt="Ugo Pattacini"/><br /><sub><b>Ugo Pattacini</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=pattacini" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/jg-rivera"><img src="https://avatars.githubusercontent.com/u/27613092?v=4?s=100" width="100px;" alt="Jose Gabrielle Rivera"/><br /><sub><b>Jose Gabrielle Rivera</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=jg-rivera" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://pdrittenhouse.com"><img src="https://avatars.githubusercontent.com/u/1556730?v=4?s=100" width="100px;" alt="P.D. Rittenhouse"/><br /><sub><b>P.D. Rittenhouse</b></sub></a><br /><a href="#ideas-pdrittenhouse" title="Ideas, Planning, & Feedback">🤔</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Daniel-Boll"><img src="https://avatars.githubusercontent.com/u/43689101?v=4?s=100" width="100px;" alt="Daniel Boll"/><br /><sub><b>Daniel Boll</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/issues?q=author%3ADaniel-Boll" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/albertschwarzkopf"><img src="https://avatars.githubusercontent.com/u/35684914?v=4?s=100" width="100px;" alt="albertschwarzkopf"/><br /><sub><b>albertschwarzkopf</b></sub></a><br /><a href="#ideas-albertschwarzkopf" title="Ideas, Planning, & Feedback">🤔</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://akulpillai.com"><img src="https://avatars.githubusercontent.com/u/32793567?v=4?s=100" width="100px;" alt="Akul Pillai"/><br /><sub><b>Akul Pillai</b></sub></a><br /><a href="#security-akulpillai" title="Security">🛡️</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/steveizzle"><img src="https://avatars.githubusercontent.com/u/45331237?v=4?s=100" width="100px;" alt="Stefan Riembauer"/><br /><sub><b>Stefan Riembauer</b></sub></a><br /><a href="#ideas-steveizzle" title="Ideas, Planning, & Feedback">🤔</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://fabriziocacicia.com"><img src="https://avatars.githubusercontent.com/u/7525888?v=4?s=100" width="100px;" alt="Fabrizio Cacicia"/><br /><sub><b>Fabrizio Cacicia</b></sub></a><br /><a href="#security-fabriziocacicia" title="Security">🛡️</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/issues?q=author%3Afabriziocacicia" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/JTunis"><img src="https://avatars.githubusercontent.com/u/11543636?v=4?s=100" width="100px;" alt="Justin Tunis"/><br /><sub><b>Justin Tunis</b></sub></a><br /><a href="#ideas-JTunis" title="Ideas, Planning, & Feedback">🤔</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=JTunis" title="Code">💻</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/issues?q=author%3AJTunis" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://mikematos84.github.io"><img src="https://avatars.githubusercontent.com/u/4093433?v=4?s=100" width="100px;" alt="Michael Matos"/><br /><sub><b>Michael Matos</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/issues?q=author%3Amikematos84" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/fatmcgav-depop"><img src="https://avatars.githubusercontent.com/u/109519102?v=4?s=100" width="100px;" alt="Gavin Williams"/><br /><sub><b>Gavin Williams</b></sub></a><br /><a href="#ideas-fatmcgav-depop" title="Ideas, Planning, & Feedback">🤔</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/msiebeneicher"><img src="https://avatars.githubusercontent.com/u/5821183?v=4?s=100" width="100px;" alt="Marc Siebeneicher"/><br /><sub><b>Marc Siebeneicher</b></sub></a><br /><a href="#ideas-msiebeneicher" title="Ideas, Planning, & Feedback">🤔</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=msiebeneicher" title="Code">💻</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/issues?q=author%3Amsiebeneicher" title="Bug reports">🐛</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=msiebeneicher" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/LuisHenri"><img src="https://avatars.githubusercontent.com/u/44511825?v=4?s=100" width="100px;" alt="Luís Henrique A. Schünemann"/><br /><sub><b>Luís Henrique A. Schünemann</b></sub></a><br /><a href="#ideas-LuisHenri" title="Ideas, Planning, & Feedback">🤔</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=LuisHenri" title="Documentation">📖</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=LuisHenri" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/george-gca"><img src="https://avatars.githubusercontent.com/u/31376482?v=4?s=100" width="100px;" alt="George"/><br /><sub><b>George</b></sub></a><br /><a href="#question-george-gca" title="Answering Questions">💬</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=george-gca" title="Documentation">📖</a> <a href="#ideas-george-gca" title="Ideas, Planning, & Feedback">🤔</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://www.linkedin.com/in/pedrorrivero/"><img src="https://avatars.githubusercontent.com/u/52980156?v=4?s=100" width="100px;" alt="Pedro Rivero"/><br /><sub><b>Pedro Rivero</b></sub></a><br /><a href="#ideas-pedrorrivero" title="Ideas, Planning, & Feedback">🤔</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/ebronson68"><img src="https://avatars.githubusercontent.com/u/111298136?v=4?s=100" width="100px;" alt="Eleanor Bronson"/><br /><sub><b>Eleanor Bronson</b></sub></a><br /><a href="#ideas-ebronson68" title="Ideas, Planning, & Feedback">🤔</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/marvin-belonio"><img src="https://avatars.githubusercontent.com/u/88089002?v=4?s=100" width="100px;" alt="Marvin Osswald"/><br /><sub><b>Marvin Osswald</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=marvin-belonio" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/dotdc"><img src="https://avatars.githubusercontent.com/u/12827900?v=4?s=100" width="100px;" alt="David Calvert"/><br /><sub><b>David Calvert</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=dotdc" title="Documentation">📖</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/issues?q=author%3Adotdc" title="Bug reports">🐛</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=dotdc" title="Code">💻</a> <a href="#maintenance-dotdc" title="Maintenance">🚧</a> <a href="#blog-dotdc" title="Blogposts">📝</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://blog.airey.be"><img src="https://avatars.githubusercontent.com/u/7644094?v=4?s=100" width="100px;" alt="Andy Airey"/><br /><sub><b>Andy Airey</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/issues?q=author%3Aaairey" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://fs02.github.io/"><img src="https://avatars.githubusercontent.com/u/3049307?v=4?s=100" width="100px;" alt="Surya Asriadie"/><br /><sub><b>Surya Asriadie</b></sub></a><br /><a href="https://github.com/AndreasAugustin/actions-template-sync/issues?q=author%3AFs02" title="Bug reports">🐛</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/jellllly420"><img src="https://avatars.githubusercontent.com/u/64725601?v=4?s=100" width="100px;" alt="jellllly420"/><br /><sub><b>jellllly420</b></sub></a><br /><a href="#ideas-jellllly420" title="Ideas, Planning, & Feedback">🤔</a> <a href="#question-jellllly420" title="Answering Questions">💬</a> <a href="https://github.com/AndreasAugustin/actions-template-sync/commits?author=jellllly420" title="Documentation">📖</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors)
specification. Contributions of any kind are welcome!

[enabling-debug-logging]: https://docs.github.com/en/actions/managing-workflow-runs/enabling-debug-logging
[deployment-keys]: https://docs.github.com/en/developers/overview/managing-deploy-keys#deploy-keys
[action-triggers]: https://docs.github.com/en/actions/reference/events-that-trigger-workflows
[template]: https://github.com/AndreasAugustin/template
[marketplace]: https://github.com/marketplace/actions/actions-template-sync
[self-usage]: https://github.com/AndreasAugustin/actions-template-sync/blob/main/.github/workflows/actions_template_sync.yml
[pr-labels]: https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels
[devto-example]: https://dev.to/andreasaugustin/github-actions-template-sync-1g9k
[github-example]: https://github.com/AndreasAugustin/teaching/blob/main/docs/git/git_action_sync.md
[github-app]: https://docs.github.com/en/developers/apps/getting-started-with-apps/about-apps#about-github-apps
[glob-pattern]: https://en.wikipedia.org/wiki/Glob_(programming)
[github-app-token]: https://github.com/tibdex/github-app-token
[dockerhub-repo]: https://hub.docker.com/r/andyaugustin/actions-template-sync
[github-repo]: https://github.com/AndreasAugustin/actions-template-sync/pkgs/container/actions-template-sync
[github-pat]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token
[other-repo-sync]: https://github.com/repo-sync/github-sync
[other-git-repo-sync]: https://github.com/it3xl/git-repo-sync
[other-action-template-repository-sync]: https://github.com/ahmadnassri/action-template-repository-sync
[git-glossary]: https://git-scm.com/docs/gitglossary
[dotdc-blog]: https://0xdc.me/blog/github-templates-and-repository-sync/
[github-create-pat]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token
[github-create-secret]: https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository
