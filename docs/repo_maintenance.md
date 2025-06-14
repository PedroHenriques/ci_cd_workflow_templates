# Template for application repository maintenance

## What these templates do

There is 1 template.

The template will:
- Syncronize the application repository with a template  repository:
  - For `.Net applications` it will sync with the [.Net template repository](https://github.com/PedroHenriques/dotnet_ms_template)
- Invoke the script, in the application repository, that will check for dependency version updates and update them

For each of the jobs the pipeline will open a pull request if there  are changes detected.

## Pre-requisites

### Scripts

These templates will interact with the following scripts in your application repository

| File path (relative to repo root) | Required | Expectation | Flags | Arguments | Example invocation |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| `cli/dependencies_update.sh` | yes | Update the version of all application dependencies, inside a Docker containers | `--cicd`<br>`-u`<br>`-y` | N/A | `sh cli/dependencies_update.sh --cicd -u -y` |

#### Detail about the script flags

**All scripts**
- `--cicd`: Signals to the script, in your application repository, that the invocation is coming from the CI/CD workflow.

**dependencies_update.sh**
- `-u`: Update all dependencies
- `-y`: Don't request confirmation before updating a dependency

### Secrets

These templates expect the following `secrets` to be configured in your application repository ([docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions))

| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `OWN_REPO_TOKEN` | Yes | A personal access token (PAT) with permissions to open pull requests in your application repository |
| `TEMPLATE_REPO_TOKEN` | Yes | A personal access token (PAT) with permissions to read the template repository.<br>More details available [here](https://github.com/marketplace/actions/actions-template-sync#3-using-a-pat) |

### Environment Variables

These templates don't require any environment variables to be configured in your application repository.

### Configuration file

These templates expect a configuration file to exist in the path `setup/workflows/.templatesyncignore`, of your application repository, with the paths to ignore for the purpose of the synchronization with the template repository.<br>
More information about the schema of this file is available [here](https://github.com/marketplace/actions/actions-template-sync#ignore-files).

## Repo maintenance template

### Inputs
| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `template_type` | Yes | The type template repository to sync with.<br>One of: `dotnet` \| `javascript_fe` |
| `pr_reviewers` | Yes | Comma separated list of pull request reviewers that should be added to any pull requests opened |

## Example of using these templates

Consider the following repo:
```
├── .github
│   ├── workflows
│   │   ├── pipeline.yml
│   │   ├── maintenance.yml
├── cli
│   ├── dependencies_update.sh
├── setup
│   ├── workflows
│   │   ├── .templatesyncignore
├── src
└── .gitignore
```

The file `.github/workflows/maintenance.yml` has the following content
```
name: maintenance
on:
  schedule:
  - cron: "0 8 * * *"
  workflow_dispatch:

jobs:
  maintenance:
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/repo_maintenance.yml@v1
    with:
      template_type: dotnet
      pr_reviewers: PedroHenriques
    secrets: inherit
```

The file `setup/workflows/.templatesyncignore` has the following content
```
src/
test/
setup/local/
*.sln
*.md
```

This will trigger the `repo_maintenance.yml` template (on the ref `v1`)
- everyday at 8 am, Github server time
- when a manual execution is requested

The behaviour for the pipeline is:
- Check if the files in the template repository have any differences to the ones in your application repository, with the exception of the files and directories marked in the file `setup/workflows/.templatesyncignore`.<br>If there are differences, a pull request will be opened in your application repository and the list of users provided as inputs will be added as reviewers
- Invoke the `cli/dependencies_update.sh` script, in your application repository.<br>If any dependency was updated, a pull request will be opened in your application repository and the list of users provided as inputs will be added as reviewers