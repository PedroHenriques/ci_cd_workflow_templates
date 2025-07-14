# Templates for applications that require building and deploying Docker images

## What these templates do

There are 2 templates, one for CI and one for CD, which are meant to be used together.

The CI template will:
- On any trigger:
  - Run static code analysis, on every trigger of the workflow
  - Run automated tests, on every trigger of the workflow
  - Run code coverage, on every trigger of the workflow
- If the trigger of the workflow is (a `pull request closed` and there was a `merge` to the specified `deployable branch`) OR (a `push event` to the specified `deployable branch`):
  - Build Docker images for the services that will be deployed
  - Push the Docker images that were built to an Azure Container Registry
  - If the optional `NUM_TAGS_TO_KEEP` environment variable is defined:
    - For services that were built: Prune the Azure Container Registry to keep, at maximum, the specified number of tags per image
    - For services that were removed: Remove from the Azure Container Registry all tags

The CD template will:
- On any trigger:
  - Builds the Kubernetes manifest files, for each service that will be deployed.<br>Handles encrypting secret manifests with [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
  - Commits the manifest files that were built to a git repository.<br>This repository should then be connected to a mechanism that will apply the manifests to the Kubernetes cluster
  - Opens a pull request, in that git repository, to the `main` branch with the manifest files that were built

## Pre-requisites

### Scripts

These templates will interact with the following scripts in your application repository

| File path (relative to repo root) | Required | Expectation | Flags | Arguments | Example invocation |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| `cli/lint.sh` | no | Run any static code analysis tools you want, inside Docker containers | `--cicd` | N/A | `sh cli/lint.sh --cicd` |
| `cli/test.sh` | no | Run automated tests on your code, inside Docker containers | `--cicd`<br>`--unit`<br>`--integration`<br>`--e2e` | N/A | `sh cli/test.sh --unit --cicd` |
| `cli/coverage.sh` | no | Generate test coverage report in lcov format, inside Docker containers | `--cicd` | N/A | `sh cli/coverage.sh --cicd` |
| `cli/build.sh` | yes | Build the Docker images for the listed services | `--cicd`<br>`--tag`<br>`--proj` | Whitespace separated list of services to build (in lowercase) | `sh cli/build.sh --cicd --tag s6d5sdf --proj myProjectName notification identity` |

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

These templates expect the following `secrets` to be configured in your application repository ([docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions))

| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `AZURE_CLIENT_ID` | Yes | A client ID with permissions to Push and Delete images to the required ACR |
| `AZURE_CLIENT_SECRET` | Yes | The client secret for the client ID |
| `AZURE_SUBSCRIPTION_ID` | Yes | The subscription ID of the ACR where the Docker images will be stored |
| `AZURE_TENANT_ID` | Yes | The tenant ID of the ACR where the Docker images will be stored |
| `IDP_REPO_TOKEN` | Yes | A personal access token (PAT) with permissions to commit and open pull requests in the git repository where the manifest files will be delivered to |

### Environment Variables

These templates expect the following `env vars` to be configured in your application repository ([docs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#creating-configuration-variables-for-a-repository))

| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `PROJECT_NAME` | Yes | The name of your project.<br>**NOTE:** Will be used as part of the Docker image names, so it must comply with the Docker image naming convention |
| `ACR_NAME` | Yes | Name of the Azure Container Registry where the Docker images will be stored |
| `ACR_REPO_NAME` | Yes | Name of the Docker image repository, in the ACR, where the push will be made to |
| `IDP_REPO_NAME` | Yes | Name of the git repository where the manifest files will be delivered to (Ex: PedroHenriques/idp_dev_cluster) |
| `AKS_RG_NAME` | Yes | Name of the Azure resource group that the Azure Kubernetes Service belongs to |
| `AKS_CLUSTER_NAME` | Yes | Name of the Azure Kubernetes Service |
| `SEALED_SECRET_CTRL_NAMESPACE` | Yes | The control namespace of the Sealed Secret installed in the AKS |
| `SEALED_SECRET_CTRL_NAME` | Yes | The control name of the Sealed Secret installed in the AKS |
| `NUM_TAGS_TO_KEEP` | No | The number of tags, per image, to keep.<br>If present will trigger a prune of the container registry to which Docker images are being pushed, in order to keep at maximum the specified number of tags |
| `APP_GITHUB_TEAM` | No | The Github team name of the application (format: Format: ORG/TEAM).<br>Will be used to ping the relevant team in PR comments. |

## CI template

### Inputs
| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `environment` | Yes | Github environment to deploy to |
| `deployable_branch_name` | Yes | Name of the branch that will trigger a deployment |
| `source_dir_name` | Yes | Name of the directory, relative to the repo's root, where the directories with source code of the service(s) are located |
| `manifest_dir_name` | Yes | Name of the directory, inside each service, where the manifest files are located |
| `custom_service_file_pattern` | Yes | File pattern, inside each service, to a file that identifies a service as a custom service |
| `build_file_pattern` | Yes | File pattern, inside each service, to a file that identifies a service needing to be built |
| `deploy_all_services_label_name` | No | Name of the PR label that signals that all deployable services should be deployed, regardless of changed files |

### Outputs
| Name | Description |
| ----------- | ----------- |
| `img_tag` | The tag of the pushed images |

## CD template

### Inputs
| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `environment` | Yes | Github environment to deploy to |
| `source_dir_name` | Yes | Name of the directory, relative to the repo's root, where the directories with source code of the service(s) are located |
| `manifest_dir_name` | Yes | Name of the directory, inside each service, where the manifest files are located |
| `custom_service_file_pattern` | Yes | File pattern, inside each service, to a file that identifies a service as a custom service |
| `build_file_pattern` | Yes | File pattern, inside each service, to a file that identifies a service needing to be built |
| `img_tag` | Yes | Tag of the Docker images of the services that will be deployed |
| `deploy_all_services_label_name` | No | Name of the PR label that signals that all deployable services should deployed, regardless of changed files |

## Manifest files

All Kubernetes manifest files can have placeholders, which will be replaced by the secrets and environment variables, defined in your application repository, with the placeholder name.  
The syntax for a placeholder is `${}`.

Besides the secrets and environment variables defined in your application repository, the following variables are also exposed by these templates and can be used in manifest files:
- `IMG_TAG`: The tag of the Docker images that will be deployed
- `IMG_NAME`: The name of the Docker image of the service being processed
- `ENVIRONMENT`: Github environment to deploy to
- `NAMESPACE`: The Kubernetes namespace
- `SERVICE`: The name of the service being processed (in lowercase)
- `BUILD_TIMESTAMP`: The epoch timestamp of the build

Example of a manifest file with placeholders:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: ${SERVICE}-config-${BUILD_TIMESTAMP}
  namespace: ${NAMESPACE}
  labels:
    app: ${SERVICE}
    env: ${ENVIRONMENT}
    version: "${IMG_TAG}"
data:
  DOTNET_ENVIRONMENT: ${DOTNET_ENVIRONMENT}
  ASPNETCORE_HTTP_PORTS: "${API_MS_PORT}"
```

## Advanced use cases

### Applications that have a custom kustomization.yaml file

There are some use cases where the application repository might have a pre-built custom `kustomization.yaml` file, for 1 or several of its services.<br>
An example of a use case where this will happen is if you want to use a Helm chart and let the Kubernetes cluster handle the process of downloading, appling and mantaintaining the charts.

This CD pipeline template will detect that the service already has a `kustomization.yaml` and will use it.<br>
You can have placeholders in your pre-built custom `kustomization.yaml` file and this CD pipeline template will replace them based on your application's repository `environment variables` and `secrets`, but it will not change any content of on the file.<br>
This means that your pre-built custom `kustomization.yaml` file must have all the `resources` declared.

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
    types: [opened, edited, reopened, synchronize, closed]

jobs:
  ci:
    uses: PedroHenriques/ci_cd_templates/.github/workflows/ci_docker.yml@v1
    with:
      environment: "dev"
      deployable_branch_name: 'main'
      source_dir_name: 'src'
      manifest_dir_name: 'Infrastructure'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'Dockerfile'
      deploy_all_services_label_name: 'deploy all services'
    secrets: inherit
  
  cd-dev:
    needs: ci
    if: ${{ github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true && github.base_ref == 'main' }}
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/cd_docker.yml@v1
    with:
      environment: "dev"
      source_dir_name: 'src'
      manifest_dir_name: 'Infrastructure'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'Dockerfile'
      img_tag: ${{ needs.ci.outputs.img_tag }}
      deploy_all_services_label_name: 'deploy all services'
    secrets: inherit
  
  cd-qa:
    needs: [ci, cd-dev]
    if: ${{ github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true && github.base_ref == 'main' }}
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/cd_docker.yml@v1
    with:
      environment: "qua"
      source_dir_name: 'src'
      manifest_dir_name: 'Infrastructure'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'Dockerfile'
      img_tag: ${{ needs.ci.outputs.img_tag }}
      deploy_all_services_label_name: 'deploy all services'
    secrets: inherit

  cd-prd:
    needs: [ci, cd-qa]
    if: ${{ github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true && github.base_ref == 'main' }}
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/cd_docker.yml@v1
    with:
      environment: "prd"
      source_dir_name: 'src'
      manifest_dir_name: 'Infrastructure'
      custom_service_file_pattern: '*.csproj'
      build_file_pattern: 'Dockerfile'
      img_tag: ${{ needs.ci.outputs.img_tag }}
      deploy_all_services_label_name: 'deploy all services'
    secrets: inherit
```

With this directory structure:
- The services are `Notification`, `Identity` and `SharedLibs`, since they are inside the `src` directory (input `source_dir_name`)
- The deployable services are `Notification`, `Identity`, since they have Kubernetes manifest files inside the manifest directory (input `manifest_dir_name`)
- The services that require building Docker images are `Notification`, `Identity`, since they have a build file (input `build_file_pattern`)
- The `SharedLibs` service is not deployable nor buildable, but is a custom service since it has a custom service file (input `custom_service_file_pattern`)

This will trigger the `ci_docker.yml` template (on the ref `v1`) when
- a `pull request` is opened, edited, reopened or synchronized or closed

The behaviour for the pipeline is:
- Changes to a buildable service, i.e., a service that has a Dockerfile (input `build_file_pattern`), will trigger the Docker image for that service to be built and pushed to the ACR
- Changes to a deployable service, i.e., a service that has manifest file(s) (input `manifest_dir_name`), will trigger the manifest files for that service to be built and a pull request opened on the IDP repository
- The `SharedLibs` service is considered a shared service, since it is not deployable and has a `.csproj` file (input `custom_service_file_pattern`), so if it has changes all deployable services will be marked for deploy (even if they had no file changes)
