# Arbiter

> Multi-AI orchestrator for independent review, structured debate, and formal go/no-go decisions inside Claude Code.

Arbiter routes tasks to Codex CLI and Gemini CLI — the two most widely installed external AI CLIs — and brings their output back into your Claude Code session. Use it to get a second opinion on a diff, run a structured panel discussion across three models, impose a formal quorum gate before a risky change, or delegate an implementation spike to an isolated worktree. Arbiter uses your existing CLI authentication; no additional API keys are required. All seven modes are invoked as slash commands.

Example: `/arbiter panel "Is a task queue the right abstraction here, or should we use a state machine?"`

## Quick Start

**Install**

```
skill7 install arbiter
```

**First use**

```
# Get an independent code review of the current diff
/arbiter review

# Ask both providers the same question in parallel
/arbiter panel "What are the failure modes of this approach?"

# Formal go/no-go gate before shipping
/arbiter quorum "Merge the payment refactor to main?"
```

## Key Features

- **Independent review** — Codex or Gemini reads your diff and returns substantive issues; style disagreements are filtered out automatically
- **True parallel panel** — both providers answer simultaneously (run_in_background), results formatted as Codex | Gemini | Claude | Consensus | Split so you can see where models agree and diverge
- **Formal quorum gate** — two-round voting (APPROVE/BLOCK/NEEDS_INFO) followed by Claude synthesis with a deterministic policy and adversarial tiebreaker
- **Adversarial claim verification** — three-round cross-check decomposes any claim into VERIFIED/CONTESTED/REJECTED verdicts
- **Isolated implementation** — delegate a spec to an external AI in a fresh git worktree; review the diff and choose merge, cherry-pick, or discard

## Privacy and Data

Arbiter invokes Codex CLI and Gemini CLI as local processes. Your diff and prompt text are sent to those providers under their own terms of service and authentication — the same as running the CLIs directly. Arbiter itself does not transmit data to any additional endpoint. No API keys are stored by Arbiter.

## Requirements

- Claude Code with skill support
- Codex CLI installed and authenticated (`codex --version`)
- Gemini CLI installed and authenticated (`gemini --version`) for modes that use Gemini
- Git repository for `review` and `implement` modes
- 120-second timeout budget per provider call

## Documentation

- [How it works](docs/how-it-works.md) — architecture, data flow, trust boundaries
- [Reference](docs/reference.md) — all commands, options, output format, troubleshooting

## Links

- [skill7.dev/plugins/arbiter](https://skill7.dev/plugins/arbiter)
- [github.com/heurema/arbiter](https://github.com/heurema/arbiter)

## License

MIT
