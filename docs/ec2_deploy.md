# Template for applications deployed directly to AWS EC2

## What this template does

This template deploys services from an application repository directly to one or more AWS EC2 targets.

Each EC2 target defines its own host, SSH user, target directory, Security Group ID, and SSH private key secret name. The workflow expands those targets into a GitHub Actions matrix and deploys the same set of changed/removed services to each target.

The template will:

- Detect which services changed by using the shared `categorize-changed-services.yml` workflow.
- Deploy only the services that need to be deployed.
- Optionally deploy all deployable services when a pull request has the configured deploy-all label.
- For each target EC2 instance, using a matrix:
  - Temporarily allow the GitHub Actions runner IP to connect to the EC2 instance over SSH.
    - If the same runner IP is already authorized in the Security Group, the workflow continues instead of failing.
  - Copy service files to the EC2 instance using `rsync`.
  - Keep the same service directory name in the EC2 target directory.
  - For each deployed service:
    - Create the target service directory on EC2 if it does not already exist.
    - Copy the required files to EC2.
    - Check if `cli/start.sh` exists in the copied service directory.
    - If `cli/start.sh` exists, make it executable and run it.
  - For each removed service:
    - Check if the service directory exists on EC2.
    - Check if `cli/stop.sh` exists in the EC2 service directory.
    - If `cli/stop.sh` exists, make it executable and run it before removal.
    - Remove the service directory from EC2.
  - Revoke the GitHub Actions runner IP from the AWS Security Group at the end of the job, even if the rule already existed before this workflow run.

### Notes

- Processes EC2 targets sequentially by default using `max-parallel: 1`.
- Does not guarantee that the deployments will follow the order of the instances you provided to this template.
- Stops deploying to remaining targets if one target fails, because the matrix uses `fail-fast: true`.

## Pre-requisites

### Repository structure

The template expects services to be direct child directories of the configured source directory.

Example:

```text
├── .github
│   ├── workflows
│   │   ├── pipeline.yml
├── src
│   ├── notification
│   │   ├── cli
│   │   │   ├── start.sh
│   │   │   ├── stop.sh
│   │   ├── .deploy
│   │   │   ├── paths
│   │   ├── Dockerfile
│   │   ├── appsettings.json
│   │   ├── Program.cs
│   ├── identity
│   │   ├── cli
│   │   │   ├── start.sh
│   │   │   ├── stop.sh
│   │   ├── Dockerfile
│   │   ├── appsettings.json
│   │   ├── Program.cs
│   ├── shared-libs
│   │   ├── Cache.cs
└── README.md
```

With this structure:

- `notification`, `identity` and `shared-libs` are services because they are direct child directories of `src`.
- Deployable services are identified by the file or directory configured in `deployment_file_name`.
- Custom/shared services are identified by the file pattern configured in `custom_service_file_pattern`.
- Services that require a build are identified by the file pattern configured in `build_file_pattern`.
- When a deployable service changes, it is copied to EC2.
- When a custom/shared service changes, the categorization workflow can mark all deployable services for deployment, depending on how `categorize-changed-services.yml` is implemented.

### Service lifecycle scripts

Each service can optionally define lifecycle scripts under its own `cli` directory.

| File path, relative to service root | Required | Expectation | Flags | Arguments | Example invocation |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| `cli/start.sh` | No | Starts or restarts the service after files have been copied to EC2 | `--cicd` | N/A | `./cli/start.sh --cicd` |
| `cli/stop.sh` | No | Stops the service before the service directory is removed from EC2 | `--cicd` | N/A | `./cli/stop.sh --cicd` |

#### Detail about the script flags

**All scripts**
- `--cicd`: Signals to the script, in your application repository, that the invocation is coming from the CI/CD workflow.

#### `cli/start.sh`

If present, the template will execute this script after copying the service files to EC2.

The script is executed from inside the service directory on EC2.

Example:

```sh
#!/bin/sh
set -e;

docker compose up -d --build;
```

#### `cli/stop.sh`

If present, the template will execute this script before removing a deleted service from EC2.

The script is executed from inside the service directory on EC2.

Example:

```sh
#!/bin/sh
set -e;

docker compose down;
```

### EC2 target directories

Each EC2 target defines its own `target_dir` inside `EC2_TARGETS_JSON`.

The template copies each service into that target directory, preserving the service directory name.

For example, if one target has:

- `source_dir_name` set to `src`
- changed service `notification`
- `target_dir` set to `/opt/apps`

Then the service will be copied on that EC2 target to:

```text
/opt/apps/notification
```

### AWS Security Group

Each EC2 target defines its own `security_group_id`.

Before deploying to a target, the workflow temporarily authorizes the current GitHub Actions runner public IPv4 address in that target's Security Group for TCP port `22`.

After that target's deployment finishes, the workflow attempts to revoke the same runner IPv4 address from that target's Security Group.

The Security Group must allow the workflow credentials to:

- Authorize ingress rules.
- Describe Security Groups.
- Revoke ingress rules.

If an identical ingress rule already exists for the same runner IPv4 address, protocol and port, the workflow treats that as non-fatal and continues.

This means that if the same rule existed before the workflow started, it can still be removed by the cleanup step. This is intentional for this template and assumes that temporary SSH access for GitHub Actions runner IPs is owned by the deployment workflow.

### SSH access

For each EC2 target, the template connects using:

- `host`
- `user`
- the SSH private key stored in the secret named by `ssh_key_secret_name`

These values are read from `EC2_TARGETS_JSON`.

The template supports two SSH host key verification modes:

1. **Pinned known hosts**, using the optional `EC2_SSH_KNOWN_HOSTS` secret.
2. **Dynamic host key discovery**, using `ssh-keyscan`.

If the optional `EC2_SSH_KNOWN_HOSTS` secret is configured, the template writes its value directly to `~/.ssh/known_hosts`.

If `EC2_SSH_KNOWN_HOSTS` is not configured, the workflow runs **ssh-keyscan** separately for each target host.

Using `EC2_SSH_KNOWN_HOSTS` is the stricter and recommended option because it pins the expected EC2 SSH host key instead of trusting the key returned during the workflow run.

The SSH user must have permissions to:

- Create directories under `target_dir`.
- Write files under `target_dir`.
- Run `cli/start.sh` and `cli/stop.sh`.
- Remove service directories under `target_dir`.

### Optional pinned SSH known hosts

For stricter SSH host verification, configure the optional `EC2_SSH_KNOWN_HOSTS` secret.

Example value:

```text
example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
```

or, when using a public IP:

```text
203.0.113.10 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
```

If multiple EC2 targets are used, the secret should contain known_hosts entries for all target hosts.

Example:

```text
203.0.113.10 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
203.0.113.11 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
```

### Required tools on the runner

The template uses the default `ubuntu-latest` GitHub-hosted runner and expects the following tools to be available:

| Tool | Usage |
| ----------- | ----------- |
| `aws` | Add and revoke the temporary Security Group SSH rule |
| `ssh` | Execute remote commands on EC2 |
| `ssh-keyscan` | Add the EC2 host key to `known_hosts` when `EC2_SSH_KNOWN_HOSTS` is not configured |
| `rsync` | Copy service files to EC2 |
| `curl` | Detect the GitHub Actions runner public IPv4 address |
| `jq` | Validate and compact the `EC2_TARGETS_JSON` matrix |

### Required tools on EC2

The EC2 instance must support:

| Tool | Usage |
| ----------- | ----------- |
| `sh` | Run service lifecycle scripts |
| `chmod` | Make lifecycle scripts executable |
| `rm` | Remove deleted service directories |

Any additional tools required by `cli/start.sh` or `cli/stop.sh`, such as `docker`, `docker compose`, `systemctl`, `pm2` or others, must also be installed on the EC2 instance.

## Secrets

These templates expect the following `secrets` to be configured in your application repository ([docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions))

| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `AWS_ACCESS_KEY_ID` | Yes | AWS access key ID used by the workflow to authorize and revoke temporary SSH access in the configured Security Group |
| `AWS_SECRET_ACCESS_KEY` | Yes | AWS secret access key used by the workflow to authorize and revoke temporary SSH access in the configured Security Group |
| Target-specific SSH private key secrets | Yes | One secret per EC2 target, referenced by `ssh_key_secret_name` inside `EC2_TARGETS_JSON` |
| `EC2_SSH_KNOWN_HOSTS` | No | Optional pinned SSH known_hosts entry or entries for all EC2 targets. When configured, the template uses this value instead of running `ssh-keyscan` |

Each EC2 target points to its SSH private key through the `ssh_key_secret_name` field.

For example, this target:

```json
{
  "name": "ec2-app-01",
  "host": "203.0.113.10",
  "user": "ubuntu",
  "target_dir": "/opt/apps",
  "security_group_id": "sg-0123456789abcdef0",
  "ssh_key_secret_name": "EC2_APP_01_SSH_PRIVATE_KEY"
}
```

requires a GitHub secret named: `EC2_APP_01_SSH_PRIVATE_KEY`.

The secret name must contain only letters, numbers and underscores, and should start with a letter or underscore.

### Environment Variables

These templates expect the following `env vars` to be configured in your application repository ([docs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#creating-configuration-variables-for-a-repository))

| Name | Required | Description |
| ----------- | ----------- | ----------- |
| `AWS_REGION` | Yes | AWS region where the EC2 instance and Security Group exist |
| `EC2_TARGETS_JSON` | Yes | JSON object describing the EC2 targets to deploy to |

#### EC2_TARGETS_JSON

`EC2_TARGETS_JSON` must be a JSON object with an `include` array. This array is used directly as the GitHub Actions matrix for the deployment job.

Example:

```json
{
  "include": [
    {
      "name": "ec2-app-01",
      "host": "203.0.113.10",
      "user": "ubuntu",
      "target_dir": "/opt/apps",
      "security_group_id": "sg-0123456789abcdef0",
      "ssh_key_secret_name": "EC2_APP_01_SSH_PRIVATE_KEY"
    },
    {
      "name": "ec2-app-02",
      "host": "203.0.113.11",
      "user": "ubuntu",
      "target_dir": "/opt/apps",
      "security_group_id": "sg-0abcdef1234567890",
      "ssh_key_secret_name": "EC2_APP_02_SSH_PRIVATE_KEY"
    }
  ]
}
```

The value can be stored as formatted JSON or as a single-line JSON string. The workflow normalizes it internally using `jq -c`.

| Field | Required | Description |
| ----------- | ----------- | ----------- |
| `name` | Yes | Logical name of the EC2 target. Used in logs and rule descriptions |
| `host` | Yes | Hostname or public IP address of the EC2 target |
| `user` | Yes | SSH username used to connect to the EC2 target |
| `target_dir` | Yes | Absolute directory path on EC2 where services will be deployed |
| `security_group_id` | Yes | AWS Security Group ID where the runner IP should be temporarily allowed for SSH access |
| `ssh_key_secret_name` | Yes | Name of the GitHub secret containing the private SSH key for this target |

## Inputs

| Name | Required | Default | Description |
| ----------- | ----------- | ----------- | ----------- |
| `environment` | Yes | N/A | Github environment to deploy to |
| `source_dir_name` | Yes | N/A | Name of the directory, relative to the repo root, where the service directories are located |
| `deployment_file_name` | Yes | N/A | Name of the file or directory, inside each service, that identifies the service as deployable |
| `custom_service_file_pattern` | Yes | N/A | File pattern, inside each service, that identifies the service as a custom/shared service |
| `build_file_pattern` | Yes | N/A | File pattern, inside each service, that identifies the service as needing to be built |
| `deploy_all_services_label_name` | No | `""` | Name of the PR label that signals that all deployable services should be deployed, regardless of changed files |
| `deploy_copy_mode` | No | `full_service` | How service files should be copied to EC2. Supported values are `full_service` and `deploy_paths` |
| `deploy_paths_file` | No | `.deploy/paths` | Relative path, inside each service, to the file containing deploy paths. Used only when `deploy_copy_mode` is `deploy_paths` |

## Deploy copy modes

The template supports two copy modes:

| Mode | Description |
| ----------- | ----------- |
| `full_service` | Copies the full service directory to EC2 |
| `deploy_paths` | Copies only the paths listed in a deploy paths file inside the service |

### `full_service` mode

When `deploy_copy_mode` is `full_service`, the full service directory is copied to EC2.

Example:

```yaml
deploy_copy_mode: full_service
```

If the changed service is `notification`, the workflow will copy:

```text
${source_dir_name}/notification/
```

to:

```text
${target_dir}/notification/
```

The copy uses `rsync --delete`, which means files that exist on EC2 but no longer exist in the source service directory will be removed.

### `deploy_paths` mode

When `deploy_copy_mode` is `deploy_paths`, the workflow reads a deploy paths file inside each service and copies only the listed paths.

Example:

```yaml
deploy_copy_mode: deploy_paths
deploy_paths_file: .deploy/paths
```

For a service named `notification`, the template will read:

```text
${source_dir_name}/notification/.deploy/paths
```

Example `.deploy/paths` file:

```text
cli
docker-compose.yml
appsettings.json
publish
```

The workflow will stage only those paths and then copy them to:

```text
${target_dir}/notification/
```

The final copy to EC2 also uses `rsync --delete`.

This means that the target service directory on EC2 will be synchronized with the staged content, not with the full source service directory.

#### Deploy paths file rules

Each deploy path must be relative to the service root.

Valid examples:

```text
cli
docker-compose.yml
config/appsettings.json
publish
```

Invalid examples:

```text
/etc/nginx/nginx.conf
../shared-file
config/../../secret
```

The template rejects deploy paths that:

- Are empty after trimming whitespace.
- Are absolute paths.
- Try to navigate outside the service directory using `..`.

Blank lines and lines starting with `#` are ignored.

Example:

```text
# Files required to run this service on EC2
cli
docker-compose.yml
publish
```

## EC2 target validation rules

Before deployment starts, the workflow validates every target in `EC2_TARGETS_JSON`.

The validation requires:

- `EC2_TARGETS_JSON` must be a JSON object.
- It must contain a non-empty `include` array.
- Each target must have non-empty string values for:
  - `name`
  - `host`
  - `user`
  - `target_dir`
  - `security_group_id`
  - `ssh_key_secret_name`
- `name` may only contain letters, numbers, `.`, `_` and `-`.
- `host` may only contain letters, numbers, `.`, `_`, `:`, and `-`.
- `user` may only contain letters, numbers, `.`, `_` and `-`.
- `target_dir` must be an absolute path.
- `target_dir` must not contain single quotes or newlines.
- `security_group_id` must start with `sg-`.
- `ssh_key_secret_name` may only contain letters, numbers and underscores.
- `ssh_key_secret_name` must start with a letter or underscore.

## Behaviour

### Deployment

When services are marked for deployment, the template will perform the deployment once per configured EC2 target:

1. Checkout the repository.
2. Validate deployment inputs.
3. Configure AWS credentials.
4. Get the GitHub Actions runner public IPv4 address.
5. Add the runner IPv4 address to the current target's configured AWS Security Group for SSH access.
6. Configure the SSH private key for the current target.
7. For each service to deploy:
   - Validate that the local service directory exists.
   - Create the target service directory on the current EC2 target.
   - Copy the service files using the configured copy mode.
   - Check if `cli/start.sh` exists on the current EC2 target.
   - If it exists, make it executable and run it.

### Deleted services

When services are marked for removal, the template performs the removal on each configured EC2 target:

1. Connect to the EC2 instance.
2. Check if the target service directory exists.
3. If the directory exists:
   - Enter the service directory.
   - Check if `cli/stop.sh` exists.
   - If it exists, make it executable and run it.
   - Remove the service directory from EC2.
4. If the service directory does not exist, skip removal.

### Temporary SSH access cleanup

The final step revokes the runner IPv4 address from the configured AWS Security Group.

This step runs with:

```yaml
if: ${{ always() && steps.ip.outputs.ipv4 != "" }}
```

This means the workflow tries to revoke the SSH rule even if a previous deployment step fails.

The workflow always attempts to revoke the runner IP at the end, including when the authorize step found that the same rule already existed.
This template assumes that temporary SSH access for GitHub Actions runner IPs is controlled by this deployment workflow. If the same Security Group rule is managed manually or by another workflow, this cleanup step can remove that rule.

## Example of using this template

Consider the following repository:

```text
├── .github
│   ├── workflows
│   │   ├── pipeline.yml
├── src
│   ├── notification
│   │   ├── cli
│   │   │   ├── start.sh
│   │   │   ├── stop.sh
│   │   ├── .deploy
│   │   │   ├── paths
│   │   ├── Dockerfile
│   │   ├── docker-compose.yml
│   │   ├── appsettings.json
│   │   ├── Program.cs
│   ├── identity
│   │   ├── cli
│   │   │   ├── start.sh
│   │   │   ├── stop.sh
│   │   ├── Dockerfile
│   │   ├── docker-compose.yml
│   │   ├── appsettings.json
│   │   ├── Program.cs
│   ├── shared-libs
│   │   ├── Cache.cs
│   │   ├── SharedLibs.csproj
└── README.md
```

The file `.github/workflows/pipeline.yml` could have the following content:

```yaml
name: pipeline

on:
  pull_request:
    types: [opened, edited, reopened, synchronize, closed]

jobs:
  ci:
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/ci_docker.yml@v1
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
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/cd_to_ec2.yml@v1
    with:
      environment: "dev"
      source_dir_name: "src"
      deployment_file_name: "docker-compose.yml"
      custom_service_file_pattern: "*.csproj"
      build_file_pattern: "Dockerfile"
      deploy_all_services_label_name: "deploy all services"
      deploy_copy_mode: "full_service"
    secrets: inherit
```

With this configuration:

- Services are read from the `src` directory.
- A service is considered deployable when it has a `docker-compose.yml` file.
- A service is considered buildable when it has a `Dockerfile`.
- A service is considered custom/shared when it matches `*.csproj`.
- When a deployable service changes, it is deployed to EC2.
- When a custom/shared service changes, the categorization workflow can mark all deployable services for deployment.
- The full service directory is copied to EC2.
- If `cli/start.sh` exists, it is executed after the copy.

## Example using deploy paths

A workflow can use `deploy_paths` mode when only specific files or directories should be sent to EC2.

```yaml
name: pipeline

on:
  pull_request:
    types: [opened, edited, reopened, synchronize, closed]

jobs:
  ci:
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/ci_docker.yml@v1
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
    uses: PedroHenriques/ci_cd_workflow_templates/.github/workflows/cd_to_ec2.yml@v1
    with:
      environment: "dev"
      source_dir_name: "src"
      deployment_file_name: "docker-compose.yml"
      custom_service_file_pattern: "*.csproj"
      build_file_pattern: "Dockerfile"
      deploy_all_services_label_name: "deploy all services"
      deploy_copy_mode: "deploy_paths"
      deploy_paths_file: ".deploy/paths"
    secrets: inherit
```

The selected GitHub environment, `dev` in this example, should define the `EC2_TARGETS_JSON` environment variable with something like:

```json
{
  "include": [
    {
      "name": "ec2-app-01",
      "host": "203.0.113.10",
      "user": "ubuntu",
      "target_dir": "/opt/apps",
      "security_group_id": "sg-0123456789abcdef0",
      "ssh_key_secret_name": "EC2_APP_01_SSH_PRIVATE_KEY"
    },
    {
      "name": "ec2-app-02",
      "host": "203.0.113.11",
      "user": "ubuntu",
      "target_dir": "/opt/apps",
      "security_group_id": "sg-0abcdef1234567890",
      "ssh_key_secret_name": "EC2_APP_02_SSH_PRIVATE_KEY"
    }
  ]
}
```

The repository or environment should also define secrets matching the configured SSH key secret names:

```text
EC2_APP_01_SSH_PRIVATE_KEY
EC2_APP_02_SSH_PRIVATE_KEY
```

For a service named `notification`, the file `src/notification/.deploy/paths` could contain:

```text
cli
docker-compose.yml
.env
publish
```

The workflow will copy only those paths to:

```text
/opt/apps/notification/
```

After the copy, it will execute:

```text
/opt/apps/notification/cli/start.sh
```

if that file exists.

## Example EC2 layout after deployment on each target

If:

- the deployed services are `notification` and `identity`

The EC2 instance will contain:

```text
/opt/apps
├── notification
│   ├── cli
│   │   ├── start.sh
│   │   ├── stop.sh
│   ├── docker-compose.yml
│   ├── appsettings.json
├── identity
│   ├── cli
│   │   ├── start.sh
│   │   ├── stop.sh
│   ├── docker-compose.yml
│   ├── appsettings.json
```

## Notes and caveats

### `rsync --delete`

Both copy modes use `rsync --delete`.

In `full_service` mode, this synchronizes the EC2 service directory with the full local service directory.

In `deploy_paths` mode, this synchronizes the EC2 service directory with the staged deploy paths only.

This is important because files that are not present in the copied source can be deleted from EC2.

### Service names

Service names come from the directory names inside the configured source directory.

The service directory name is preserved on EC2.

Example:

```text
src/notification
```

is deployed to:

```text
${target_dir}/notification
```

### Lifecycle script execution directory

Both lifecycle scripts are executed from inside the service directory on EC2.

For example:

```sh
cd "${target_dir}/${service}";
./cli/start.sh --cicd;
```

This allows scripts to use relative paths based on the service root.
