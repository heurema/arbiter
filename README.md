# Arbiter

> Multi-AI orchestrator for independent review, structured debate, formal go/no-go decisions, and parallel implementation exploration inside Claude Code.

Arbiter routes tasks to Codex CLI and Gemini CLI — the two most widely installed external AI CLIs — and brings their output back into your Claude Code session. Use it to get a second opinion on a diff, run a structured panel discussion across three models, impose a formal quorum gate before a risky change, delegate an implementation spike to an isolated worktree, or generate three independent implementations in parallel and pick the best one. Arbiter uses your existing CLI authentication; no additional API keys are required. All eight modes are invoked as slash commands.

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

# Generate 3 independent implementations, compare, and merge the best one
/arbiter diverge "add rate limiting middleware to the Express app"

# Design-level only — explore approaches before writing any code
/arbiter diverge --lite "redesign the auth module"

# Run existing tests against all three solutions before evaluating
/arbiter diverge --run-tests "refactor the OrderService class"
```

## Key Features

- **Independent review** — Codex or Gemini reads your diff and returns substantive issues; style disagreements are filtered out automatically
- **True parallel panel** — both providers answer simultaneously (run_in_background), results formatted as Codex | Gemini | Claude | Consensus | Split so you can see where models agree and diverge
- **Formal quorum gate** — two-round voting (APPROVE/BLOCK/NEEDS_INFO) followed by Claude synthesis with a deterministic policy and adversarial tiebreaker
- **Adversarial claim verification** — three-round cross-check decomposes any claim into VERIFIED/CONTESTED/REJECTED verdicts
- **Isolated implementation** — delegate a spec to an external AI in a fresh git worktree; review the diff and choose merge, cherry-pick, or discard
- **Parallel diverge** — generate 3 independent implementations in isolated worktrees using different strategy hints (minimal / refactor / redesign), evaluate with an anonymized decision matrix, and merge only the solution you choose; lite mode produces design docs instead of code

## Privacy and Data

Arbiter invokes Codex CLI and Gemini CLI as local processes. Your diff and prompt text are sent to those providers under their own terms of service and authentication — the same as running the CLIs directly. Arbiter itself does not transmit data to any additional endpoint. No API keys are stored by Arbiter.

## Requirements

- Claude Code with skill support
- Codex CLI installed and authenticated (`codex --version`)
- Gemini CLI installed and authenticated (`gemini --version`) for modes that use Gemini
- Git repository for `review`, `implement`, and `diverge` modes
- Clean working tree for `diverge` (uncommitted changes must be stashed or committed first)
- 120-second timeout budget per provider call for most modes; `diverge` defaults to 300 seconds per agent (configurable via `--timeout`)

## Documentation

- [How it works](docs/how-it-works.md) — architecture, data flow, trust boundaries
- [Reference](docs/reference.md) — all commands, options, output format, troubleshooting

## Links

- [skill7.dev/plugins/arbiter](https://skill7.dev/plugins/arbiter)
- [github.com/heurema/arbiter](https://github.com/heurema/arbiter)

## License

MIT
