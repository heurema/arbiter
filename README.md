# Arbiter

<div align="center">

**Route decisions through independent AI reviewers**

[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-5b21b6?style=flat-square)](https://skill7.dev/plugins/arbiter)
[![Version](https://img.shields.io/badge/version-0.3.0-5b21b6?style=flat-square)](https://github.com/heurema/arbiter)
[![License MIT](https://img.shields.io/badge/license-MIT-5b21b6?style=flat-square)](LICENSE)

```bash
claude plugin marketplace add heurema/emporium
claude plugin install arbiter@emporium
```

</div>

## What it does

Code review, architecture decisions, and risky merges all suffer from the same bias: a single reviewer. Arbiter routes tasks to Codex CLI and Gemini CLI, brings their output back into your Claude Code session, and lets Claude synthesize the results. It handles everything from a quick second opinion on a diff to a formal two-round quorum vote with an adversarial tiebreaker. Arbiter uses your existing CLI authentication — no additional API keys required.

## Install

<!-- INSTALL:START — auto-synced from emporium/INSTALL_REFERENCE.md -->
```bash
claude plugin marketplace add heurema/emporium
claude plugin install arbiter@emporium
```
<!-- INSTALL:END -->

<details>
<summary>Manual install from source</summary>

```bash
git clone https://github.com/heurema/arbiter
cd arbiter
claude plugin install .
```

</details>

## Quick start

```bash
# Get an independent code review of the current diff
/arbiter review

# Ask both providers the same question in parallel
/arbiter panel "What are the failure modes of this approach?"

# Formal go/no-go gate before shipping
/arbiter quorum "Merge the payment refactor to main?"
```

## Commands

| Command | Description |
|---------|-------------|
| `/arbiter review` | Sends the current diff to Codex or Gemini and returns substantive issues; filters style disagreements automatically |
| `/arbiter panel "<question>"` | Runs both providers in parallel and formats results as Codex \| Gemini \| Claude \| Consensus \| Split |
| `/arbiter quorum "<decision>"` | Two-round voting (APPROVE/BLOCK/NEEDS_INFO) with Claude synthesis and adversarial tiebreaker |
| `/arbiter verify "<claim>"` | Three-round cross-check that decomposes a claim into VERIFIED/CONTESTED/REJECTED verdicts |
| `/arbiter implement "<spec>"` | Delegates a spec to an external AI in a fresh git worktree; returns a diff to review |
| `/arbiter diverge "<spec>"` | Generates 3 independent implementations in isolated worktrees using different strategy hints, evaluated via anonymized decision matrix |
| `/arbiter diverge --lite "<spec>"` | Design-only mode — produces design docs instead of code |
| `/arbiter diverge --run-tests "<spec>"` | Runs existing tests against all three solutions before evaluation |

## Features

- Independent review routes to Codex or Gemini and filters noise before surfacing findings
- True parallel panel runs both providers simultaneously via `run_in_background` so neither influences the other
- Formal quorum gate enforces a deterministic policy with an adversarial tiebreaker when models split
- Adversarial claim verification decomposes any statement into per-claim verdicts across three rounds
- Isolated implementation creates a fresh git worktree per provider so changes never touch your working tree
- Parallel diverge produces three independent solutions with different strategy hints (minimal / refactor / redesign) and lets you merge only the one you choose

## Requirements

- Claude Code with skill support
- Codex CLI installed and authenticated (`codex --version`)
- Gemini CLI installed and authenticated (`gemini --version`) for modes that use Gemini
- Git repository for `review`, `implement`, and `diverge` modes
- Clean working tree for `diverge` (uncommitted changes must be stashed or committed first)
- 120-second timeout per provider call for most modes; `diverge` defaults to 300 seconds per agent (configurable via `--timeout`)

## Privacy

Arbiter invokes Codex CLI and Gemini CLI as local processes. Your diff and prompt text are sent to those providers under their own terms of service and authentication — the same as running the CLIs directly. Arbiter itself does not transmit data to any additional endpoint. No API keys are stored by Arbiter.

## See also

- [skill7.dev/plugins/arbiter](https://skill7.dev/plugins/arbiter) — plugin page and changelog
- [github.com/heurema/emporium](https://github.com/heurema/emporium) — plugin registry
- [docs/how-it-works.md](docs/how-it-works.md) — architecture, data flow, trust boundaries
- [docs/reference.md](docs/reference.md) — all commands, options, output format, troubleshooting

## License

[MIT](LICENSE)
