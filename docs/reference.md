# Arbiter — Reference

## Commands

### `/arbiter review`

Sends the current git diff to an external AI for an independent code review. Codex is the default provider. Pass the diff to Gemini instead by piping.

**Options**
- `--gemini` — use Gemini CLI instead of Codex
- `--with-diff` — always include diff even if the provider already sees it from context (useful for `ask`)

**Output**
A structured review with: Summary, Issues (severity: critical/major/minor), Suggestions. Style disagreements are filtered and not reported.

**Example**
```
/arbiter review
/arbiter review --gemini
```

---

### `/arbiter ask "question"`

Gets an external AI opinion on a question. Both Codex and Gemini perspectives are presented when relevant context differs. Use `--with-diff` to include the current diff as context.

**Options**
- `--with-diff` — attach current git diff to the prompt

**Output**
Provider answer(s) with attribution. If both providers were queried, answers are shown side by side.

**Examples**
```
/arbiter ask "Is this the right place to add caching?"
/arbiter ask --with-diff "Will this change affect backward compatibility?"
```

---

### `/arbiter implement "spec"`

Delegates an implementation task to an external AI working inside an isolated git worktree. When done, Arbiter shows the resulting diff and offers three choices: merge into the current branch, cherry-pick specific commits, or discard the worktree.

**Output**
Git diff of changes made by the external AI, followed by a merge/cherry-pick/discard prompt.

**Examples**
```
/arbiter implement "Add rate limiting middleware to the Express app, 100 req/min per IP"
/arbiter implement "Write unit tests for the OrderService class"
```

**Note:** The implementation runs in an isolated worktree. Your working tree is not modified until you explicitly choose to merge or cherry-pick.

---

### `/arbiter panel "question"`

Sends the same question to Codex and Gemini simultaneously (true parallel, `run_in_background`), then Claude adds its own answer. Results are rendered as a five-column comparison.

**Output format**
```
| Codex | Gemini | Claude | Consensus | Split |
```
Consensus is populated when all three agree. Split lists the points of divergence.

**Examples**
```
/arbiter panel "Should we use optimistic or pessimistic locking here?"
/arbiter panel "What are the security implications of this token storage approach?"
```

---

### `/arbiter quorum "proposal"`

Formal two-round go/no-go gate.

**Round 1:** Codex and Gemini each vote independently: APPROVE, BLOCK, or NEEDS_INFO with reasoning.

**Round 2:** Each provider sees the other's vote and reasoning, then may revise.

**Synthesis:** Claude applies the deterministic policy:
- Unanimous APPROVE → go
- Any BLOCK → no-go
- Split or NEEDS_INFO → escalate with summary of concerns

An adversarial tiebreaker is applied when votes are split after round 2.

**Examples**
```
/arbiter quorum "Merge the payment refactor to main?"
/arbiter quorum "Remove the feature flag for experiment-42 from production?"
```

---

### `/arbiter verify "claim"`

Three-round adversarial verification.

**Round 1:** Each provider answers independently.
**Round 2:** Claims are compared; contested points are identified.
**Round 3:** Each provider challenges the other's contested claims.

**Output:** Per-claim verdict — VERIFIED, CONTESTED, or REJECTED — with supporting evidence and the source of disagreement.

**Examples**
```
/arbiter verify "This migration is safe to run without downtime"
/arbiter verify "The new index will reduce query time from O(n) to O(log n)"
```

---

### `/arbiter continue "follow-up"`

Resumes the last Gemini CLI session with a follow-up message. Useful for iterating on a previous `ask` or `implement` result without losing context.

**Note:** Codex session continuation is not supported in v1. Only Gemini sessions can be resumed.

**Example**
```
/arbiter continue "Can you refactor that to use a generator instead?"
```

---

### `/arbiter diverge "spec"`

Generates three independent implementations in parallel, each in an isolated git worktree with a different strategy hint, then scores all three with an anonymized evaluator and presents a Decision Matrix. Use when a non-trivial spec has multiple valid approaches and you want to explore the solution space before committing.

**Requires a clean working tree.** Uncommitted changes must be stashed or committed first.

**Options**
- `--lite` — design-level only; agents produce design documents (approach, files, risks, dependencies) instead of code. No worktrees are created. Approximately 10× faster than full diverge. Cannot be combined with `--run-tests`.
- `--strategies "a,b,c"` — custom strategy hints injected verbatim into each agent's prompt. If fewer than 3 are provided, defaults fill the remainder. If more than 3, the first 3 are used.
- `--providers "claude,codex"` — restrict which providers are used. Fewer than 3 providers triggers a warning; strategies are redistributed.
- `--timeout <seconds>` — per-agent timeout in seconds. Default is 300. Exit code 124 means the agent timed out.
- `--run-tests` — run the project's existing test suite against each worktree before evaluation. Solutions that fail existing tests have their Correctness score capped. Cannot be combined with `--lite`.

**Default strategies**

| Strategy | Hint | Default provider |
|----------|------|-----------------|
| `minimal` | Smallest possible change; minimize lines, files, and new dependencies | Claude (sonnet subagent) |
| `refactor` | Improve existing structure; extract helpers, rename for clarity, improve testability | Codex (GPT) |
| `redesign` | Best possible abstraction; may introduce new modules or patterns | Gemini |

**Output format — Decision Matrix**

```
## Diverge Report: <spec summary>

Evaluator: independent (anonymized labels, formula-computed totals)
Run ID: 20260301T100000Z-12345

### Summary

|                | Solution 1 | Solution 2 | Solution 3 |
|----------------|-----------|-----------|-----------|
| Strategy       | minimal   | refactor  | redesign  |
| Files changed  | 3         | 5         | 8         |
| Lines +/-      | +42/-12   | +87/-34   | +156/-67  |
| New deps       | 0         | 0         | 1         |
| Tests          | PASS      | PASS      | 2 FAIL    |

### Decision Matrix

| Criterion (weight)    | Sol 1 | Sol 2 | Sol 3 | Notes |
|-----------------------|-------|-------|-------|-------|
| Correctness (5)       | 4     | 4     | 5     | ...   |
| Change size (3)       | 5     | 3     | 1     | ...   |
| Maintainability (4)   | 3     | 5     | 4     | ...   |
| Test coverage (4)     | 3     | 4     | 5     | ...   |
| Backward compat (3)   | 5     | 4     | 3     | ...   |
| Performance (2)       | 4     | 4     | 4     | ...   |
| Security (5)          | 4     | 4     | 4     | ...   |
| Weighted total        | 99    | 106   | 104   | formula-computed |

### Recommendation
Solution 2 (refactor) scores highest. Solution 1 is the safe fallback.

### Options
1. Merge Solution 1 → current branch
2. Merge Solution 2 → current branch
3. Merge Solution 3 → current branch
4. Inspect full diff (specify solution number)
5. Hybrid: cherry-pick files manually
6. Discard all
```

After the user selects, Arbiter reveals which provider produced each solution.

**Examples**
```
/arbiter diverge "add rate limiting middleware to the Express app, 100 req/min per IP"
/arbiter diverge --lite "redesign the auth module"
/arbiter diverge --run-tests "refactor the OrderService class"
/arbiter diverge --strategies "functional,oop,event-driven" "rewrite the event bus"
/arbiter diverge --timeout 600 "implement full-text search"
/arbiter diverge --providers "claude,codex" "add input validation"
```

**Note:** A budget warning is shown when the spec is very short (under 50 words) and involves few context files. Diverge costs significantly more than a single `implement` call; confirm before proceeding or switch to `implement` for simple tasks.

---

## Configuration

Arbiter reads no configuration file. All behavior is controlled by the mode and inline options. Provider selection follows these defaults:

| Mode | Default provider | Alternate |
|------|-----------------|-----------|
| review | Codex | `--gemini` flag |
| ask | Codex | automatic (both for panel-style questions) |
| implement | Codex | — |
| panel | Both | — |
| quorum | Both | — |
| verify | Both | — |
| continue | Gemini only | — |
| diverge | All three (Claude, Codex, Gemini) | `--providers` to restrict |

---

## Output Format

All multi-provider modes prefix each section with the provider name for traceability:

```
[Codex] ...
[Gemini] ...
[Claude] ...
[Synthesis] ...
```

Single-provider modes omit the prefix and return results directly.

---

## Troubleshooting

**"Binary not found: codex"**
Install Codex CLI from [github.com/openai/codex](https://github.com/openai/codex) and ensure it is on your `$PATH`. Run `codex --version` to verify.

**"Binary not found: gemini"**
Install Gemini CLI from [github.com/google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli) and ensure it is on your `$PATH`. Run `gemini --version` to verify.

**Authentication error**
Run the provider CLI manually to re-authenticate (`codex auth` or `gemini auth`). Arbiter uses whichever credentials the CLI already has.

**Timeout after 120 seconds**
The provider took too long to respond. For large diffs, try `--gemini` for `review` (Gemini handles large context well via chunking). For `implement`, break the spec into smaller tasks.

**Large diff warning (Gemini)**
Diffs over 300 lines are automatically chunked and sent to Gemini sequentially. This increases latency but prevents context overflow errors. No action required.

**`continue` says "no previous session"**
Gemini CLI session state is maintained by the CLI itself. If you started a new terminal session or the CLI state was cleared, there is no session to resume. Start a new `ask` or `implement` instead.

**`diverge` says "Working tree has uncommitted changes"**
Diverge requires all three worktrees to start from the same commit. Stash your changes (`git stash push -m "diverge: pre-run stash"`), run diverge, then restore with `git stash pop`. Alternatively, commit the changes first and re-run.

**`diverge` provider unavailable or authentication error**
If one provider fails to authenticate or is unavailable, diverge redistributes its strategy to an available provider and continues with two providers in two worktrees. If all providers fail, the mode stops and suggests using `arbiter implement` instead. Re-authenticate the failing provider with `codex auth` or `gemini login`.

**`diverge` agent timed out (exit code 124)**
The agent exceeded the per-agent timeout (default 300 seconds). Use `--timeout <seconds>` to increase the limit, or break the spec into smaller tasks. The timed-out solution is excluded from the Decision Matrix.

**`diverge` says "WARNING: HEAD moved during diverge"**
Another process committed to the current branch while the agents were running. The merge may produce conflicts. Inspect `git log` to understand what changed, then confirm or abort the merge. If you abort, the worktrees are cleaned up automatically; the uncommitted changes in each worktree are discarded.

**`diverge` evaluator overflow**
When three full diffs exceed the evaluator's context limit, diverge falls back to evaluating solutions using only stats and self-assessment cards (without diff content). Scores will be less precise. For large specs, use `--lite` to explore designs first, then delegate the chosen design to `arbiter implement`.

**Orphan worktrees from a crashed diverge run**
If diverge crashed without running cleanup, stale worktree registrations may remain. Run `git worktree list | grep arbiter-diverge` to find them, then `git worktree remove --force <path>` for each. Running `git worktree prune` removes registrations for worktrees whose directories no longer exist.
