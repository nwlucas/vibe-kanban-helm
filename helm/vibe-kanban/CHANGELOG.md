# Changelog

All notable changes to the Vibe Kanban Helm chart are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.0.3] - 2026-03-31

### Added

- Worker sidecar containers support via `worker.extraContainers`.
- Extra pod volumes for worker via `worker.extraVolumes`.
- Extra volume mounts on the main worker container via `worker.extraVolumeMounts`.
- Example `git-sync` sidecar that periodically pulls repos under `/home/node`.

### Changed

- Updated worker image to `0.1.37`.
- Updated relay image to `relay-v0.1.7`.
- Updated remote server image to `remote-v0.1.26`.

## [0.0.2] - 2025-03-31

### Added

- GitHub Actions CI/CD pipeline for building Docker images.
- Helm chart packaging and publishing workflow.

### Changed

- Disabled resource limits by default for easier local development.
- Updated worker image to `0.1.25`.

## [0.0.1] - 2025-03-30

### Added

- Initial Helm chart with PostgreSQL, ElectricSQL, server, relay, and worker components.
- NSA-hardened pod security contexts (non-root, dropped capabilities, seccomp).
- Optional relay/tunnel server support.
- Worker pre-start init script support.
- NetworkPolicy with default-deny and explicit allow rules.
- PodDisruptionBudget for the server.
- Ingress resources for server, relay, and worker.
- Secret management with existing secret or chart-managed secret options.
