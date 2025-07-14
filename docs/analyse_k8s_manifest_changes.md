# Templates for analysing PRs with kubernetes manifest file changes and automatically approving those PRs based on cluster resource impact

## What these templates do

There is 1 template.

The template will:
- Loop through each modified file in the pull request and for each one will:
  - Check if the file is a kubernetes manifest that was added or modified
  - Check if any of the configured properties, that affect cluster resources, were changed
- After analysing all files:
  - If no cluster resource affecting changes were made, this template will approve the pull request and leave a comment pinging the application team (based on the provided inputs to the template)
  - If any cluster resource affecting changes were made, this template will leave a comment piging the approval team (based on the provided inputs to the template) and leave a report of the relevant changes

## Pre-requisites

### Github application team name

This template assumes that the name of the application team, related to the pull request that was opened, is available in the pull request title between single quotes.
**Ex:** For a pull request with the title `Deploy for the project ci_cd_test and team 'PedroHenriques/ci_cd_test'`, this template will assume the application Github team name is `PedroHenriques/ci_cd_test`

### Secrets

These templates expect the following `secrets` to be configured in your application repository ([docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions))

| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `OWN_REPO_TOKEN` | Yes | A personal access token (PAT) with permissions to approve and comment in pull requests in the manifest repository |

#### `OWN_REPO_TOKEN` permissions:

The token must have the following permissions:

| Scope              | Purpose                                      |
| ------------------ | -------------------------------------------- |
| `repo`             | Full access to private repositories          |
| `public_repo`      | (Optional) If only working with public repos |
| `write:discussion` | Allows writing comments on PRs               |
| `read:org`         | If you use org-level repo access (optional)  |

## Analyse K8s manifest changes template

### Inputs
| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `approval_team_name` | Yes | GitHub Team to ping in case of approval needed (Format: ORG/TEAM) |

## Example of using these templates

Consider the following repo:
```
├── .github
│   ├── workflows
│   │   ├── pipeline.yml
├── base
│   ├── app
│   │   ├── ci_cd_test
│   │   │   ├── Api
│   │   │   │   ├── config-map.yml
│   │   │   │   ├── deployment.yml
│   │   │   │   ├── ...
│   │   │   ├── kustomization.yaml
│   │   ├── kustomization.yaml
└── .gitignore
```

The file `.github/workflows/pipeline.yml` has the following content
```
name: pipeline
on:
  pull_request:
    types: [opened, edited, reopened, synchronize]

permissions:
  pull-requests: write

jobs:
  ci:
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/analyse_k8s_manifest_changes.yml@v1
    with:
      approval_team_name: PedroHenriques/DEVOPS
    secrets: inherit
```

This will trigger the `analyse_k8s_manifest_changes.yml` template (on the ref `v1`) when
- a `pull request` is opened, edited, reopened or synchronized

The behaviour for the pipeline is described [above](#What-these-templates-do).
