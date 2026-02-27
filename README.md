# arbiter

Multi-AI orchestrator — dispatch to Codex CLI and Gemini CLI for review, ask, implement, and panel modes.

Replaces [codex-partner](https://github.com/heurema/codex-partner) with multi-provider support.

## Installation

<!-- INSTALL:START — auto-synced from emporium/INSTALL_REFERENCE.md -->
```bash
claude plugin marketplace add heurema/emporium
claude plugin install arbiter@emporium
```
<!-- INSTALL:END -->

## Prerequisites

At least one provider CLI must be installed:

- **Codex CLI**: [github.com/openai/codex](https://github.com/openai/codex)
- **Gemini CLI**: [github.com/google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli)

## Usage

```
/arbiter                                → help + provider status
/arbiter review                         → code review (codex)
/arbiter review --via gemini            → code review (gemini)
/arbiter review --base main             → PR review
/arbiter review --commit abc123         → specific commit review
/arbiter review "focus on SQL injection" → custom review instructions
/arbiter ask "question"                 → ask codex
/arbiter ask --via gemini "question"    → ask gemini
/arbiter ask --with-diff "question"     → include git diff as context
/arbiter implement "spec"               → delegate to codex in worktree
/arbiter implement --via gemini "spec"  → delegate to gemini in worktree
/arbiter panel "question"               → ask both, compare answers
/arbiter continue "follow-up"           → resume last gemini session
```

## Modes

| Mode | Default Provider | Description |
|------|-----------------|-------------|
| review | codex | Independent code review |
| ask | codex | Get a second opinion |
| implement | codex | Delegate work to isolated worktree |
| panel | both | Ask all providers, compare |
| continue | gemini | Resume last session (gemini only in v1) |

## License

MIT
