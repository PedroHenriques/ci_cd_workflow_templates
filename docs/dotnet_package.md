# Templates for applications that require building and deploying .Net packages to a package manager

## What these templates do

There is 1 template.

The CI template will:
- Leave a comment on the pull request explaining how to signal a deployment.<br>If the trigger of the workflow is a `pull request opened`
- Run static code analysis, on every trigger of the workflow
- Run automated tests, on every trigger of the workflow
- Run code coverage, on every trigger of the workflow
- If the trigger of the workflow is a `pull request closed` and there was a `merge` to the specified `deployable branch`:
  - Checks if the pull request had the label assigned to a `major`, `minor` or `patch` version update
  - If it had any of those tags:
    - Build the packages for the services that will be deployed
    - Bump the version of the services that will be deployed
    - Publish the packages to Nuget
    - Commit and push to the specified `deployable branch` the updated project files with the new package version

## Pre-requisites

### Scripts

These templates will interact with the following scripts in your application repository

| File path (relative to repo root) | Required | Expectation | Flags | Arguments | Example invocation |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| `cli/lint.sh` | no | Run any static code analysis tools you want, inside Docker containers | `--cicd` | N/A | `sh cli/lint.sh --cicd` |
| `cli/test.sh` | no | Run automated tests on your code, inside Docker containers | `--cicd`<br>`--unit`<br>`--integration`<br>`--e2e` | N/A | `sh cli/test.sh --unit --cicd` |
| `cli/coverage.sh` | no | Generate test coverage report in lcov format, inside Docker containers | `--cicd` | N/A | `sh cli/coverage.sh --cicd` |

#### Detail about the script flags

**All scripts**
- `--cicd`: Signals to the script, in your application repository, that the invocation is coming from the CI/CD workflow.

**test.sh**
- `--unit`: Run all relevant unit tests, if any
- `--integration`: Run all relevant integration tests, if any
- `--e2e`: Run all relevant end-to-end tests, if any

### Secrets

These templates expect the following `secrets` to be configured in your application repository ([docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions))

| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `OWN_REPO_TOKEN` | Yes | A personal access token (PAT) with permissions to commit and comment in pull requests and to push to the deployable branches in your application repository |
| `NUGET_TOKEN` | Yes | A token to the Nuget account where the packages will be published to |

## CI template

### Inputs
| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `environment` | Yes | Github environment to deploy to |
| `deployable_branch_name` | Yes | Name of the branch that will trigger a deployment |
| `source_dir_name` | Yes | Name of the directory, relative to the repo's root, where the directories with source code of the service(s) are located |
| `deployment_file-or-dir_path` | Yes | Path, inside each service, that identifies the service as a deployable service |
| `custom_service_file_pattern` | Yes | File pattern, inside each service, to a file that identifies a service as a custom service |
| `build_file_pattern` | Yes | File pattern, inside each service, to a file that identifies a service needing to be built |
| `major_version_label_name` | Yes | Name of the PR label that signals that a deployment should be made with a MAJOR version bump |
| `minor_version_label_name` | Yes | Name of the PR label that signals that a deployment should be made with a MINOR version bump |
| `patch_version_label_name` | Yes | Name of the PR label that signals that a deployment should be made with a PATCH version bump |
| `deploy_all_services_label_name` | No | Name of the PR label that signals that all deployable services should be deployed, regardless of changed files |

## Example of using these templates

Consider the following repo:
```
├── .github
│   ├── workflows
│   │   ├── pipeline.yml
├── cli
│   ├── test.sh
├── src
│   ├── package1
│   │   ├── Services
│   │   │   ├── Myservice.cs
│   │   ├── Program.cs
│   │   ├── build.txt
│   │   ├── package1.csproj
│   ├── package2
│   │   ├── Program.cs
│   │   ├── build.txt
│   │   ├── package2.csproj
│   ├── sharedLibs
│   │   ├── SomeService.cs
│   │   ├── sharedLibs.csproj
└── .gitignore
```

The file `.github/workflows/pipeline.yml` has the following content
```
name: pipeline
on:
  pull_request:
    types: [opened, edited, reopened, synchronize, closed]

jobs:
  ci:
    uses: PedroHenriques/ci_cd_test_templates/.github/workflows/ci_dotnet_package.yml@v1
    with:
      environment: "dev"
      deployable_branch_name: 'main'
      source_dir_name: 'src'
      deployment_file-or-dir_path: 'build.txt'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'build.txt'
      major_version_label_name: 'major'
      minor_version_label_name: 'minor'
      patch_version_label_name: 'patch'
      deploy_all_services_label_name: 'deploy all services'
    secrets: inherit
```

With this directory structure:
- The services are `package1`, `package2` and `sharedLibs`, since they are inside the `src` directory (input `source_dir_name`)
- The deployable services are `package1` and `package2`, since they have the deployment file inside their directories (input `deployment_file-or-dir_path`)
- The services that require building a package are `package1` and `package2`, since they have a build file (input `build_file_pattern`)
- The `sharedLibs` service is not deployable nor buildable, but is a custom service since it has a custom service file (input `custom_service_file_pattern`)

This will trigger the `ci_dotnet_package.yml` template (on the ref `v1`) when
- a `pull request` is opened, edited, reopened, synchronized or closed

The behaviour for the pipeline is:
- Changes to a deployable service (input `deployment_file-or-dir_path`) that has a build file (input `build_file_pattern`) will trigger the package for that service to be built and published to Nuget
- The `sharedLibs` service is considered a shared service, since it is not deployable and has a `*.csproj` file (input `custom_service_file_pattern`), so if it has changes all deployable services will be marked for deploy (even if they had no file changes)

**NOTE:** In this example we use an empty file named `build.txt` to identify the services to build and deploy.