# Arbiter — How It Works

## Overview

Arbiter is a single-skill Claude Code plugin. The entire implementation lives in `arbiter/SKILL.md`, which Claude Code loads as a skill and executes as structured instructions. There is no separate daemon, server, or compiled binary. When you invoke `/arbiter <mode>`, Claude Code activates the skill, parses the mode and arguments, and orchestrates one or more external CLI calls.

## Components

**arbiter/SKILL.md** — the skill file. Defines all eight modes, their invocation patterns, output schemas, error handling rules, and synthesis logic. This is the single source of truth for Arbiter's behavior.

**Codex CLI** — external process invoked via shell. Used as the default provider for `review`, `ask`, `implement`, `quorum`, `verify`, `panel`, and as one of three agents in `diverge`. Codex receives the prepared prompt or diff and returns text on stdout.

**Gemini CLI** — external process invoked via shell. Used as the secondary provider in `panel`, `quorum`, `verify`, and for `review` via diff pipe. For `continue`, Gemini is the only provider because it maintains session state; Codex does not support session continuation in v1. In `diverge`, Gemini runs as the third agent in its own isolated worktree.

**Claude (sonnet subagent)** — launched by the orchestrator as a background subagent in `diverge` mode. Claude acts as the first of three parallel implementers, working in its own isolated worktree with the `minimal` strategy hint by default. The orchestrator and the subagent are separate invocations; the evaluator that scores solutions is a third, separate Claude invocation to mitigate self-preference bias.

**Git** — used by `review` (to produce the diff), `implement` (to create and manage an isolated worktree), and `diverge` (to create three isolated detached worktrees and merge the selected solution).

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
         ├─ multi-provider modes (panel, quorum, verify)
         │    ├─ Codex CLI call  (run_in_background: true)
         │    └─ Gemini CLI call (run_in_background: true)
         │             │
         │             ▼
         │    wait for both → Claude synthesizes → structured output
         │
         └─ diverge mode
              ├─ preflight: clean tree check, worktree prune
              ├─ freeze spec + context into Problem Pack (RUN_DIR)
              ├─ create 3 isolated detached worktrees (A, B, C)
              ├─ write strategy-hint prompt files per worktree
              ├─ dispatch 3 agents in parallel (run_in_background: true):
              │    ├─ Agent A: Claude sonnet subagent (minimal)
              │    ├─ Agent B: Codex CLI (refactor)
              │    └─ Agent C: Gemini CLI (redesign)
              ├─ staged evaluation:
              │    ├─ Pass 1: stats + self-assessment cards (anonymized as Sol 1/2/3)
              │    └─ Pass 2: full diff on demand (after selection)
              ├─ separate evaluator subagent scores all 3 solutions
              │    (anonymized labels, formula-computed weighted totals)
              ├─ present Decision Matrix → user selects
              └─ merge selected solution → cleanup all worktrees
```

For `panel`, both calls are dispatched simultaneously using `run_in_background: true`. The skill waits for both to complete before rendering the comparison table.

For `quorum`, there are two rounds. Round 1: independent votes. Round 2: each provider sees the other's vote and may revise. Claude then applies the deterministic policy (unanimous APPROVE → go; any BLOCK → no-go; NEEDS_INFO → escalate) with an adversarial tiebreaker for split votes.

For `verify`, there are three rounds: independent answers, claim comparison to identify contested points, and adversarial cross-check where each provider challenges the other's contested claims. Output is per-claim verdicts.

For `diverge`, all three agents run as background tasks simultaneously. Agents do not commit — all changes remain as uncommitted modifications in the worktree. The evaluator is a separate Claude invocation that receives anonymized solution labels (Solution 1/2/3) with no provider names, to prevent model-preference bias. Weighted totals are computed via bash arithmetic, not by the LLM.

## Trust Boundaries

Arbiter's trust boundary is the boundary of your existing Codex CLI and Gemini CLI installations.

- Arbiter does not call any API directly.
- Diff content and prompt text travel to the respective AI provider via the same channel as invoking the CLI manually.
- No data is stored by Arbiter. No telemetry is collected.
- Arbiter does not write to disk except when `implement` or `diverge` creates worktrees (standard git behavior).

You should apply the same data-handling policy to Arbiter that you apply to running Codex CLI or Gemini CLI directly.

### Diverge-Specific Trust Boundaries

`diverge` introduces additional local disk activity that does not involve additional network calls beyond the existing provider dispatch:

- **Worktrees are local.** Three git worktrees are created in a temporary directory (`$TMPDIR/arbiter-diverge.*`) with owner-only permissions (`chmod 700`). They are automatically removed by a `trap EXIT` handler when the skill completes.
- **Problem Pack is local.** The frozen spec and context files are written to the same `$RUN_DIR` temporary directory. The pack is never transmitted anywhere beyond the provider CLIs.
- **No extra network calls.** Diverge dispatches to the same three providers (Claude subagent, Codex CLI, Gemini CLI) via the same channels used by other modes. No additional endpoints are contacted.
- **Audit artifacts are local.** The diverge report (`.dev/diverge-report.md`) and metrics log (`.dev/diverge-metrics.jsonl`) are written to the repository working tree. They are not transmitted externally.
- **Evaluator is a local subagent.** The anonymized evaluation pass is a separate Claude Code subagent invocation — it does not contact any external service beyond the Claude API your session already uses.

## Error Handling

| Condition | Behavior |
|-----------|----------|
| Binary not found (`codex`, `gemini`) | Skill reports which binary is missing and exits cleanly |
| Authentication error from provider | Raw error surfaced to user with suggested fix |
| Timeout (>120s per provider) | Call is aborted; partial results (if any) are discarded |
| Large diff (>300 lines) for Gemini | Diff is split into chunks and sent sequentially |
| Worktree already exists | `implement` prompts before overwriting |
| Dirty working tree (diverge) | Mode stops; user is offered stash / commit / abort |
| Provider unavailable during diverge | Strategy redistributed to an available provider in a separate worktree |
| All diverge providers fail | Mode stops with errors; suggests `arbiter implement` as fallback |
| HEAD drift during diverge merge | Warning shown; user must confirm merge or abort |
| Evaluator context overflow (diverge) | Falls back to stats and self-assessment cards only (no diff content) |

## Limitations

- `continue` resumes only Gemini sessions. Codex session continuation is not supported in v1.
- `implement` delegates work to an external AI; the resulting diff must be reviewed and merged manually — Arbiter never auto-merges.
- Parallel calls in `panel`, `quorum`, and `verify` count against both providers' rate limits simultaneously.
- `diverge` requires a clean working tree. Uncommitted changes must be stashed or committed before running.
- `diverge` runs three agents in parallel; it counts against all three providers' rate limits simultaneously and has a higher token cost than single-provider modes. A budget warning is shown for trivial specs.
- `diverge --lite --run-tests` is not allowed: lite mode produces design documents, not code, so there is nothing to test.
- All modes require the respective CLI to be installed, authenticated, and reachable in `$PATH`.
