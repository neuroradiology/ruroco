# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2024-08-04

### Added

- implement passing IP address as env var (`RUROCO_IP`) to commands executed by commander

### Changed

- code refactoring
- add auto formatting and linting
- add auto create releases
- increase test speed
- fix coverage warnings
- add/update docs
- add code coverage

## [0.1.2] - 2024-06-16

### Fixed

- Fix server crashing after first UDP packet received

## [0.1.1] - 2024-06-16

### Fixed

- Fix client command binding to 127.0.0.1 instead of binding to 0.0.0.0 when sending UDP packet to host

## [0.1.0] - 2024-06-09

### Added

- Initial Release

[0.2.0]: https://github.com/beac0n/ruroco/compare/v0.1.2..v0.2.0

[0.1.2]: https://github.com/beac0n/ruroco/compare/v0.1.1..v0.1.2

[0.1.1]: https://github.com/beac0n/ruroco/compare/v0.1.0..v0.1.1

<!-- generated by git-cliff -->