# Templates for applications that require building and deploying Docker images

## What these templates do

There are 2 templates, one for CI and one for CD, which are meant to be used together.

The CI template will:
- Run static code analysis, on every trigger of the workflow
- Run automated tests, on every trigger of the workflow
- Run code coverage, on every trigger of the workflow
- Build Docker images for the services that will be deployed, if the trigger of the workflow is a `push event` to the specified `deployable branch`
- Push the Docker images that were built to an Azure Container Registry, if the trigger of the workflow is a `push event` to the specified `deployable branch`

The CD template will:
- Builds the Kubernetes manifest files, for each service that will be deployed  
Handles encrypting secret manifests with [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- Commits the manifest files that were built to a git repository. This repository should then be connected to a mechanism that will apply the manifests to the Kubernetes cluster
- Opens a pull request, in that git repository, to the `main` branch with the manifest files that were built

## Pre-requisites

### Scripts

These templates will interact with the following scripts in your application repository

| File path (relative to repo root) | Required | Expectation | Flags | Arguments | Example invocation |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| `cli/lint.sh` | no | Run any static code analysis tools you want, inside Docker containers | `--cicd` | N/A | `sh cli/lint.sh --cicd` |
| `cli/test.sh` | no | Run automated tests on your code, inside Docker containers | `--cicd`<br>`--unit`<br>`--integration`<br>`--e2e` | N/A | `sh cli/test.sh --unit --cicd` |
| `cli/coverage.sh` | no | Generate test coverage report in lcov format, inside Docker containers | `--cicd` | N/A | `sh cli/coverage.sh --cicd` |
| `cli/build.sh` | yes | Build the Docker images for the listed services | `--cicd`<br>`--tag`<br>`--proj` | Whitespace separated list of services to build | `sh cli/build.sh --cicd --tag s6d5sdf --proj myProjectName notification identity` |

#### Detail about the script flags

**All scripts**
- `--cicd`: Signals to the script, in your application repository, that the invocation is coming from the CI/CD workflow.

**test.sh**
- `--unit`: Run all relevant unit tests, if any
- `--integration`: Run all relevant integration tests, if any
- `--e2e`: Run all relevant end-to-end tests, if any

**build.sh**
- `--tag`: The tag to be used on the Docker images being built
- `--proj`: The name of the project (used in the Docker image names)

### Docker image names

All Docker images built should comply with the following naming convention: `<project name>_<service name>:<image tag>`
- `<project name>`: The name of your project. Sent to the `build.sh` script in the `--proj` flag
- `<service name>`: The name of the service associated with the Docker image
- `<image tag>`: The tag for the Docker image. Sent to the `build.sh` script in the `--tag` flag

### Secrets

These templates require the following `secrets` to be configured in your application repository ([docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions))
- `AZURE_CLIENT_ID`: A client ID with permissions to Push images to the required ACR
- `AZURE_CLIENT_SECRET`: The client secret for the client ID
- `AZURE_SUBSCRIPTION_ID`: The subscription ID of the ACR where the Docker images will be pushed to
- `AZURE_TENANT_ID`: The tenant ID of the ACR where the Docker images will be pushed to
- `IDP_REPO_TOKEN`: A personal access token (PAT) with permissions to commit and open pull requests in the git repository where the manifest files will be delivered to

### Environment Variables

These templates require the following `env vars` to be configured in your application repository ([docs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#creating-configuration-variables-for-a-repository))
- `PROJECT_NAME`: The name of your project. **NOTE:** Will be used as part of the Docker image names, so it must comply with the Docker image naming convention
- `ACR_NAME`: Name of the Azure Container Registry where the Docker images will be pushed to
- `ACR_REPO_NAME`: Name of the Docker image repository, in the ACR, where the push will be made to
- `IDP_REPO_NAME`: Name of the git repository where the manifest files will be delivered to (Ex: PedroHenriques/idp_dev_cluster)
- `AKS_RG_NAME`: Name of the Azure resource group that the Azure Kubernetes Service belongs to
- `AKS_CLUSTER_NAME`: Name of the Azure Kubernetes Service
- `SEALED_SECRET_CTRL_NAMESPACE`: The control namespace of the Sealed Secret installed in the AKS
- `SEALED_SECRET_CTRL_NAME`: The control name of the Sealed Secret installed in the AKS

## CI template

### Inputs
- `environment`: (**Required**) Github environment to deploy to
- `deployable_branch_name`: (**Required**) Name of the branch that will trigger a deployment
- `source_dir_name`: (**Required**) Name of the directory, relative to the repo's root, where the directories with source code of the service(s) are located
- `manifest_dir_name`: (**Required**) Name of the directory, inside each service, where the manifest files are located
- `custom_service_file_pattern`: (**Required**) File pattern, inside each service, to a file that identifies a service as a custom service
- `build_file_pattern`: (**Required**) File pattern, inside each service, to a file that identifies a service needing to be built

### Outputs
- `img_tag`: The tag of the pushed images

## CD template

### Inputs
- `environment`: (**Required**) Github environment to deploy to
- `source_dir_name`: (**Required**) Name of the directory, relative to the repo's root, where the directories with source code of the service(s) is located
- `manifest_dir_name`: (**Required**) Name of the directory, inside each service, where the manifest files are located
- `custom_service_file_pattern`: (**Required**) File pattern, inside each service, to a file that identifies a service as a custom service
- `build_file_pattern`: (**Required**) File pattern, inside each service, to a file that identifies a service needing to be built
- `img_tag`: (**Required**) Tag of the Docker images of the services that will be deployed

## Manifest files

All Kubernetes manifest files can have placeholders, which will be replaced by the secrets and environment variables, defined in your application repository, with the placeholder name.  
The syntax for a placeholder is `${}`.

The following variables are exposed by these templates and can be used in manifest files:
- `IMG_TAG`: The tag of the Docker images that will be deployed
- `IMG_NAME`: The name of the Docker image of the service being processed
- `ENVIRONMENT`: Github environment to deploy to
- `NAMESPACE`: The Kubernetes namespace
- `SERVICE`: The name of the service being processed

Example of a manifest file with placeholders:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: ${SERVICE}-config
  namespace: ${NAMESPACE}
  labels:
    app: ${SERVICE}
    env: ${ENVIRONMENT}
    version: ${IMG_TAG}
data:
  DOTNET_ENVIRONMENT: ${DOTNET_ENVIRONMENT}
  ASPNETCORE_HTTP_PORTS: ${API_MS_PORT}
```

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
│   ├── Notification
│   │   ├── Infrastructure
│   │   │   ├── deployment.yml
│   │   │   ├── secret.yml
│   │   ├── Dockerfile
│   │   ├── Notification.csproj
│   │   ├── Program.cs
│   ├── Identity
│   │   ├── Infrastructure
│   │   │   ├── deployment.yml
│   │   │   ├── secret.yml
│   │   ├── Dockerfile
│   │   ├── Identity.csproj
│   │   ├── Program.cs
│   ├── SharedLibs
│   │   ├── Cache.cs
│   │   ├── SharedLibs.csproj
└── .dockerignore
└── .gitignore
└── my_project.sln
```

The file `.github/workflows/pipeline.yml` has the following content
```
name: pipeline
on:
  pull_request:
    types: [opened, edited, reopened, synchronize]
  push:
    branches:
      - 'main'
jobs:
  ci:
    uses: PedroHenriques/ci_cd_templates/.github/workflows/ci_docker.yml@v1.0.0
    with:
      environment: "dev"
      deployable_branch_name: 'main'
      source_dir_name: 'src'
      manifest_dir_name: 'Infrastructure'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'Dockerfile'
    secrets: inherit
  
  cd-dev:
    needs: ci
    if: ${{ github.event_name == 'push' && github.ref_name == 'main' }}
    uses: PedroHenriques/ci_cd_templates/.github/workflows/cd_docker.yml@v1.0.0
    with:
      environment: "dev"
      source_dir_name: 'src'
      manifest_dir_name: 'Infrastructure'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'Dockerfile'
      img_tag: ${{ needs.ci.outputs.img_tag }}
    secrets: inherit
  
  cd-qa:
    needs: [ci, cd-dev]
    if: ${{ github.event_name == 'push' && github.ref_name == 'main' }}
    uses: PedroHenriques/ci_cd_templates/.github/workflows/cd_docker.yml@v1.0.0
    with:
      environment: "qua"
      source_dir_name: 'src'
      manifest_dir_name: 'Infrastructure'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'Dockerfile'
      img_tag: ${{ needs.ci.outputs.img_tag }}
    secrets: inherit

  cd-prd:
    needs: [ci, cd-qa]
    if: ${{ github.event_name == 'push' && github.ref_name == 'main' }}
    uses: PedroHenriques/ci_cd_templates/.github/workflows/cd_docker.yml@v1.0.0
    with:
      environment: "prd"
      source_dir_name: 'src'
      manifest_dir_name: 'Infrastructure'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'Dockerfile'
      img_tag: ${{ needs.ci.outputs.img_tag }}
    secrets: inherit
```

With this directory structure:
- The services are `Notification`, `Identity` and `SharedLibs`, since they are inside the `src` directory (input `source_dir_name`)
- The deployable services are `Notification`, `Identity`, since they have Kubernetes manifest files inside the manifest directory (input `manifest_dir_name`)
- The services that require building Docker images are `Notification`, `Identity`, since they have a build file (input `build_file_pattern`)
- The `SharedLibs` service is not deployable nor buildable, but is a custom service since it has a custom service file (input `custom_service_file_pattern`)

This will trigger the `ci_docker.yml` (on the tag `v1.0.0`) template when
- a `pull request` is opened, edited, reopened or synchronized
- a `push` is made to the branch `main`

The behaviour for the pipeline is:
- Changes to a deployable service (input `manifest_dir_name`) that has a Dockerfile (input `build_file_pattern`) will trigger the Docker image for that service to be built and pushed to the ACR
- The `SharedLibs` service is considered a shared service, since it is not deployable and has a `.csproj` file (input `custom_service_file_pattern`), so if it has changes all deployable services will be marked for deploy (even if they had no file changes)
