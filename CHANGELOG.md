# Changelog

All notable changes to this project will be documented in this file.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Configurable Claude model routing for diverge implementation and evaluator stages
- Reliable nested Claude CLI invocation guidance using sanitized auth environment
- Task-specific model configuration examples for Claude and Gemini
- Verified Gemini 3 headless routing strings documented: `auto-gemini-3`, `gemini-3.1-pro-preview`, and `gemini-3-flash-preview`

### Fixed
- Documentation now consistently describes provider config as applying to Claude, Codex, and Gemini
- Reference docs no longer incorrectly claim that Arbiter has no configuration file

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
