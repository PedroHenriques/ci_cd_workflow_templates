# Templates for applications that require building and deploying Javascript packages to a package manager

## What these templates do

There is 1 template.

The CI template will:
- Leave a comment on the pull request explaining how to signal a deployment, if the trigger of the workflow is a `pull request opened`
- Run static code analysis, on every trigger of the workflow
- Run automated tests, on every trigger of the workflow
- Run code coverage, on every trigger of the workflow
- If the trigger of the workflow is a `pull request closed` and there was a `merge` to the specified `deployable branch`:
  - Checks if the pull request had the label assigned to a `major`, `minor` or `patch` version update
  - If it had any of those tags:
    - Build the packages for the services that will be deployed
    - Bump the version of the services that will be deployed
    - Publish the packages to NPM
    - Commit and push to the specified `deployable branch` the updated files with the new package version

## Pre-requisites

### Scripts

These templates will interact with the following scripts in your application repository

| File path (relative to repo root) | Required | Expectation | Flags | Arguments | Example invocation |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| `cli/lint.sh` | no | Run any static code analysis tools you want, inside Docker containers | `--cicd` | N/A | `sh cli/lint.sh --cicd` |
| `cli/test.sh` | no | Run automated tests on your code, inside Docker containers | `--cicd`<br>`--unit`<br>`--integration`<br>`--e2e` | N/A | `sh cli/test.sh --unit --cicd` |
| `cli/coverage.sh` | no | Generate test coverage report in lcov format, inside Docker containers | `--cicd` | N/A | `sh cli/coverage.sh --cicd` |
| `cli/build.sh` | yes | Build the packages for the listed services | `--cicd`<br>`--tag`<br>`--proj` | Whitespace separated list of services to build | `sh cli/build.sh --cicd --tag s6d5sdf --proj myProjectName notification identity` |

#### Detail about the script flags

**All scripts**
- `--cicd`: Signals to the script, in your application repository, that the invocation is coming from the CI/CD workflow.

**test.sh**
- `--unit`: Run all relevant unit tests, if any
- `--integration`: Run all relevant integration tests, if any
- `--e2e`: Run all relevant end-to-end tests, if any

**build.sh**
- `--tag`: The type of version bump (major, minor or patch) that will be applied
- `--proj`: The name of the project

### Secrets

These templates require the following `secrets` to be configured in your application repository ([docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions))
- `OWN_REPO_TOKEN`: A personal access token (PAT) with permissions to commit and comment in pull requests in your application repository
- `NPM_TOKEN`: A token to the NPM account where the packages will be published to

### Environment Variables

These templates require the following `env vars` to be configured in your application repository ([docs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#creating-configuration-variables-for-a-repository))
- `PROJECT_NAME`: The name of your project.

## CI template

### Inputs
- `environment`: (**Required**) Github environment to deploy to
- `deployable_branch_name`: (**Required**) Name of the branch that will trigger a deployment
- `source_dir_name`: (**Required**) Name of the directory, relative to the repo's root, where the directories with source code of the service(s) are located
- `deployment_file-or-dir_path`: (**Required**) Path, inside each service, that identifies the service as a deployable service
- `custom_service_file_pattern`: (**Required**) File pattern, inside each service, to a file that identifies a service as a custom service
- `build_file_pattern`: (**Required**) File pattern, inside each service, to a file that identifies a service needing to be built
- `major_version_label_name`: (**Required**) Name of the PR label that signals that a deployment should be made with a MAJOR version bump
- `minor_version_label_name`: (**Required**) Name of the PR label that signals that a deployment should be made with a MINOR version bump
- `patch_version_label_name`: (**Required**) Name of the PR label that signals that a deployment should be made with a PATCH version bump

## Example of using these templates

Consider the following repo:
```
├── .github
│   ├── workflows
│   │   ├── pipeline.yml
├── cli
│   ├── build.sh
│   ├── test.sh
├── src
│   ├── package1
│   │   ├── js
│   │   │   ├── index.js
│   │   ├── webpack.config.js
│   │   ├── package.json
│   ├── package2
│   │   ├── js
│   │   │   ├── index.js
│   │   ├── package.json
│   ├── sharedLibs
│   │   ├── index.js
└── .npmignore
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
    uses: PedroHenriques/ci_cd_templates/.github/workflows/ci_js_package.yml@v1
    with:
      environment: "dev"
      deployable_branch_name: 'main'
      source_dir_name: 'src'
      deployment_file-or-dir_path: 'package.json'
      custom_service_file_pattern: 'index.js'
      build_file_pattern: 'webpack.config.js'
      major_version_label_name: 'major'
      minor_version_label_name: 'minor'
      patch_version_label_name: 'patch'
    secrets: inherit
```

With this directory structure:
- The services are `package1`, `package2` and `sharedLibs`, since they are inside the `src` directory (input `source_dir_name`)
- The deployable services are `package1` and `package2`, since they have the deployment file inside their directories (input `deployment_file-or-dir_path`)
- The services that require building a package are `package1` and `package2`, since they have a build file (input `build_file_pattern`)
- The `sharedLibs` service is not deployable nor buildable, but is a custom service since it has a custom service file (input `custom_service_file_pattern`)

This will trigger the `ci_js_package.yml` (on the tag `v1`) template when
- a `pull request` is opened, edited, reopened, synchronized or closed

The behaviour for the pipeline is:
- Changes to a deployable service (input `deployment_file-or-dir_path`) that has a build file (input `build_file_pattern`) will trigger the package for that service to be built and published to NPM
- The `sharedLibs` service is considered a shared service, since it is not deployable and has an `index.js` file (input `custom_service_file_pattern`), so if it has changes all deployable services will be marked for deploy (even if they had no file changes)
