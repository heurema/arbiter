# Arbiter — How It Works

## Overview

Arbiter is a single-skill Claude Code plugin. The entire implementation lives in `arbiter/SKILL.md`, which Claude Code loads as a skill and executes as structured instructions. There is no separate daemon, server, or compiled binary. When you invoke `/arbiter <mode>`, Claude Code activates the skill, parses the mode and arguments, and orchestrates one or more external CLI calls.

## Components

**arbiter/SKILL.md** — the skill file. Defines all seven modes, their invocation patterns, output schemas, error handling rules, and synthesis logic. This is the single source of truth for Arbiter's behavior.

**Codex CLI** — external process invoked via shell. Used as the default provider for `review`, `ask`, `implement`, `quorum`, `verify`, and `panel`. Codex receives the prepared prompt or diff and returns text on stdout.

**Gemini CLI** — external process invoked via shell. Used as the secondary provider in `panel`, `quorum`, `verify`, and for `review` via diff pipe. For `continue`, Gemini is the only provider because it maintains session state; Codex does not support session continuation in v1.

**Git** — used by `review` (to produce the diff) and `implement` (to create and manage an isolated worktree).

## Data Flow

```
User → /arbiter <mode> [args]
         │
         ▼
   Claude Code (skill activated)
         │
         ├─ prepare prompt or diff
         │
         ├─ single provider modes (review, ask, implement, continue)
         │    └─ shell call → Codex CLI or Gemini CLI → stdout → Claude parses
         │
         └─ multi-provider modes (panel, quorum, verify)
              ├─ Codex CLI call  (run_in_background: true)
              └─ Gemini CLI call (run_in_background: true)
                       │
                       ▼
              wait for both → Claude synthesizes → structured output
```

For `panel`, both calls are dispatched simultaneously using `run_in_background: true`. The skill waits for both to complete before rendering the comparison table.

For `quorum`, there are two rounds. Round 1: independent votes. Round 2: each provider sees the other's vote and may revise. Claude then applies the deterministic policy (unanimous APPROVE → go; any BLOCK → no-go; NEEDS_INFO → escalate) with an adversarial tiebreaker for split votes.

For `verify`, there are three rounds: independent answers, claim comparison to identify contested points, and adversarial cross-check where each provider challenges the other's contested claims. Output is per-claim verdicts.

## Trust Boundaries

Arbiter's trust boundary is the boundary of your existing Codex CLI and Gemini CLI installations.

- Arbiter does not call any API directly.
- Diff content and prompt text travel to the respective AI provider via the same channel as invoking the CLI manually.
- No data is stored by Arbiter. No telemetry is collected.
- Arbiter does not write to disk except when `implement` creates a worktree (standard git behavior).

You should apply the same data-handling policy to Arbiter that you apply to running Codex CLI or Gemini CLI directly.

## Error Handling

| Condition | Behavior |
|-----------|----------|
| Binary not found (`codex`, `gemini`) | Skill reports which binary is missing and exits cleanly |
| Authentication error from provider | Raw error surfaced to user with suggested fix |
| Timeout (>120s per provider) | Call is aborted; partial results (if any) are discarded |
| Large diff (>300 lines) for Gemini | Diff is split into chunks and sent sequentially |
| Worktree already exists | `implement` prompts before overwriting |

## Limitations

- `continue` resumes only Gemini sessions. Codex session continuation is not supported in v1.
- `implement` delegates work to an external AI; the resulting diff must be reviewed and merged manually — Arbiter never auto-merges.
- Parallel calls in `panel`, `quorum`, and `verify` count against both providers' rate limits simultaneously.
- All modes require the respective CLI to be installed, authenticated, and reachable in `$PATH`.
