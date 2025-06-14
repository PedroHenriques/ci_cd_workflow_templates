# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## 2025-05-27

### Added

- Reusable template for .Net packages  (**Refs**: `main` and `v1`)

- Reusable template for repository maintenance, that will perform 2 jobs:  (**Refs**: `main` and `v1`)
  - Sync an application repo with a template repo
  - Run project dependency version updates

- Support for the application to contain a kustomization.yaml file for a specific service, which will be used instead of the default template (**Refs**: `main` and `v1`)

## 2025-02-19

### Changed

- Changed templates to honor she bang (#!) terminal declration (**Refs**: `main` and `v1`)

## 2025-02-18

### Added

- Support for pruning the container registry, if the `NUM_TAGS_TO_KEEP` environment variable is defined in the application repository (**Refs**: `main` and `v1`)

## 2025-02-13

### Added

- Initial version of the "ci docker", "cd docker" and "ci js package" templates
- Ref for these templates on "main" and "v1"