# CI/CD Github workflow templates

## Purpose

This repo contains a set of Github workflow templates that can be used in your application repository without you having to worry about the details of how to make everything work.

There are several workflow templates available, based on the build artifacts that your application will generate and where those artifacts should be deployed to.

**The sales pitch:**
- Abstract the complexity of a modern CI/CD workflow so that you can focus on writing code
- Support several artifact types and deployment destinations
- These templates work on mono repos and poli repos
- Define a common interface between the CI/CD pipeline and your repository
- Optimize the CI/CD workflow for you
- Allow some customization

## How to use these templates

Follow these steps:

1. [Read the pre-requisites section](#pre-requisites)
2. [Choose the template for your use case](#available-templates)
3. Configure your application repository with the secrets, environment variables and files required by the chosen template
4. Add a pipeline configuration to your application repository that references the chosen template (examples in the documentation of each template)
5. Run your application repository pipeline and validate that everything is working

## Pre-requisites

In order to use these templates your application repository needs to comply with the following schema:

### `cli` directory

This directory, at the root of your repository, will contain shell scripts that the templates will invoke at various stages of the CI/CD pipeline.

More information on the expected scripts in each template's [documentation](#available-templates).

### Services

One of the crucial steps in all the CI/CD templates is determining what changed in the "git diff" that triggered the workflow.  
This allows for:
- Running code checks and tests only for the services that changed
- Building and publishing artifacts only for the services that changed
- Deploying only the services that changed

All these templates will categorize the changed services in your application repository as:
- **Deployable Service**: A service that will be deployed to the remote target
- **Buildable Service**: A service for which a build artifact will be built
- **Removed Service**: A service that was removed  from the  application

Services can belong in multiple categories.<br>
Example:
- A service that has a Dockerfile and Kubernetes manifest files is both "buildable" and "deployable"
- A service that only has Kubernetes manifest files is only "deployable"

Services are identified by a directory inside the "source directory", which is an input of the templates.  
Inside each service directory you can place all the files needed to build and deploy that service.

**Requirements:**
- The name of service directories cannot contain whitespaces

Some templates might have additional requirements (stated in the template's [documentation](#available-templates))

**Example**

Assuming a repository with the following directory structure
```
├── src
│   ├── Notification
│   │   ├── Models
│   │   │   ├── *
│   ├── Identity
│   │   ├── Models
│   │   │   ├── *
│   ├── SharedLibs
│   │   ├── *
├── node_modules
├── package.json
├── package-lock.json
└── .gitignore
```
If you use a template and specify the directory `src` as the "source directory" for this repository, then the workflow templates will identify the following services: `Notification`, `Identity` and `SharedLibs`.

## Available templates

| Build Artifact Type | When to use | Versions | Documentation |
| ----------- | ----------- | ----------- | ----------- |
| Docker | Use these templates if your application requires building and deploying Docker images | `@main`<br>`@v1` | [doc](/docs/docker.md) |
| Javascript Package | Use these templates if your application requires building and deploying JS packages to a package manager | `@main`<br>`@v1` | [doc](/docs/js_package.md) |
| .Net Package | Use these templates if your application requires building and deploying .Net packages to a package manager | `@main`<br>`@v1` | [doc](/docs/dotnet_package.md) |
| Repository maintenance | Use this template to run maintenance jobs on application repositories | `@main`<br>`@v1` | [doc](/docs/repo_maintenance.md) |