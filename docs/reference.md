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
