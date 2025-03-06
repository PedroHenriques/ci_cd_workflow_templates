# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## 2025-03-06

### Added

- Docker templates: Trigger on pull request closed with merge to deployable branch (**Refs**: `main` and `v1`)
- Docker + JS Package templates: Support for triggering a deployment of all services, regarless of changed files, when a specific PR label is set (**Refs**: `main` and `v1`)
- Docker CD template: Sintax (kubeconform) and security (trivy) validations for generated manifest files (**Refs**: `main` and `v1`)
- Docker CD template: Base64 encoding of secrets added to the secrets manifest file, before encryption. Secrets defined in the application repository no longer need to be stored as base64 (**Refs**: `main` and `v1`)
- Docker templates: Handling of removed services - Removes services from Kustomization.yml files + deletes all image tags from the container registry for these services (**Refs**: `main` and `v1`)
- Explicit logout from Azure even if the pipeline exits with an error (**Refs**: `main` and `v1`)

### Changed

- JS Package template: Moved NodeJS setup to before the build artifact step, which allows the application repository's build.sh script to use node and NPM (**Refs**: `main` and `v1`)

### Fixed

- JS package template: Now passing the --cicd flag to the build.sh script (**Refs**: `main` and `v1`)

## 2025-02-19

### Changed

- Changed templates to honor she bang (#!) terminal declration (**Refs**: `main` and `v1`)

## 2025-02-18

### Added

- Support for pruning the container registry, if the `NUM_TAGS_TO_KEEP` environment variable is defined in the application repository (**Refs**: `main` and `v1`)

## 2025-02-13

### Added

- Initial version of the "ci docker", "cd docker" and "ci js package" templates (**Refs**: `main` and `v1`)
