# Changelog

All notable changes to this project will be documented in this file.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.3.0] - 2026-03-01

### Added
- Doctor mode — dependency check and configuration validation
- PARTIAL_SUCCESS run summary
- Diverge mode — 3 independent implementations, compare and select

### Fixed
- Remove `timeout` command dependency (unavailable on macOS)

## [0.2.0] - 2026-02-28

### Added
- Verify mode
- Quorum mode — 2-provider consensus gate for high-stakes decisions
- Parallel execution in panel mode

### Fixed
- Use `run_in_background` for true parallel panel execution
- Remove `timeout` binary from quorum (macOS)

## [0.1.0] - 2026-02-27

### Added
- Initial release — panel mode with Codex and Gemini providers
