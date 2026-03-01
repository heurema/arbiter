---
name: arbiter
description: >
  Use when user says "/arbiter", "/codex", "ask codex", "ask gemini",
  "codex review", "gemini review", "panel mode", "compare AIs",
  "second opinion from codex", "second opinion from gemini",
  "quorum", "go/no-go", "consensus vote",
  "verify", "fact-check", "cross-check", "is this true",
  or wants to delegate work to an external AI CLI (Codex, Gemini).
  Arbiter dispatches to multiple providers and presents compared results.
---

# Arbiter — Multi-AI Orchestrator

## Team Model
- **User**: domain knowledge, final decisions
- **Claude Code**: orchestrator, deep codebase understanding, planning
- **Codex CLI**: independent verification (GPT), implementation, code review
- **Gemini CLI**: alternative perspective, session continuity, yolo implementation

## Providers

### Routing Table

| Mode | Codex Command | Gemini Command | Default |
|------|--------------|----------------|---------|
| ask | `codex exec --ephemeral -C "$PWD" -p fast --output-last-message "$OUT" "<prompt>"` | `gemini -p "<prompt>" -o text` | codex |
| review | `codex review [flags]` | `git diff \| gemini -p "<review_prompt>" -o text` | codex |
| implement | `codex exec --full-auto -C "<worktree>" "<spec>"` | `(cd "<worktree>" && gemini -p "<spec>" -y -o text)` | codex |
| continue | Not supported in v1 | `gemini -r latest -p "<follow-up>" -o text` | gemini |

**Note:** `quorum` and `verify` are not in this table — they are multi-provider protocols, not single-provider dispatch. Quorum always runs both providers (see [quorum mode](#quorum)). Verify always runs both providers (see [verify mode](#verify)).

### Capability Notes
- **Codex** uses `-p` for profiles (fast, default, full). `-c model_reasoning_effort=high` for reasoning override.
- **Gemini** uses `-o text` always (not `-o json` — that wraps in transport envelope). `-y` for auto-accept in implement.
- **Codex review** outputs plain text, never JSON. Always parse as human-readable.
- **Gemini stderr** is noisy — always capture to tempfile, parse for errors, discard noise.

## Command Surface

```
/arbiter review                              → codex review (uncommitted)
/arbiter review --via gemini                 → gemini review via diff pipe
/arbiter review --base main                  → PR review
/arbiter review --commit abc123              → specific commit review
/arbiter review "focus on SQL injection"     → custom instructions
/arbiter ask "question"                      → default provider (codex)
/arbiter ask --via gemini "question"         → explicit provider
/arbiter ask --with-diff "question"          → include git diff as context
/arbiter implement "spec"                    → codex (default)
/arbiter implement --via gemini "spec"       → gemini in yolo mode
/arbiter panel "question"                    → ask BOTH providers, compare
/arbiter quorum "question or proposal"       → 2-provider vote + Claude synthesis
/arbiter quorum --with-diff "question"       → include git diff as context
/arbiter verify "question or claim"          → 3-round cross-verification
/arbiter verify --with-diff "question"       → include git diff as context
/arbiter continue "follow-up"               → gemini only (codex not supported in v1)
/arbiter                                     → show help + provider status
```

## Input Parsing

Parse the user's `/arbiter` invocation by matching these patterns in order:

1. Extract mode: first word after `/arbiter` → `review|ask|implement|panel|quorum|verify|continue`
2. Extract `--via <provider>`: if present, set provider (codex|gemini). Not applicable to `quorum` (always both).
3. Extract mode-specific flags:
   - review: `--base <branch>`, `--commit <sha>`
   - ask: `--with-diff`
   - quorum: `--with-diff`
   - verify: `--with-diff`
4. Remaining quoted string → prompt/question/spec
5. If no mode → show help

**Conflict rules:**
- `--base` + `--commit` → error: "Choose one: --base or --commit"
- `--via unknown` → error: "Unknown provider. Available: codex, gemini"
- `quorum --via <any>` → error: "Quorum always runs both providers. No --via flag."
- `verify --via <any>` → error: "Verify always runs both providers. No --via flag."
- `continue --via codex` → error: "Continue not supported for codex in v1. Use --via gemini."

## Modes

### review

**Purpose:** Independent code review from an external AI.

**Flow:**
1. Check we're in a git repo. If not → stop with message.
2. Determine diff source:
   - Default: uncommitted changes (`git diff --stat` + `git diff --cached --stat`)
   - `--base <branch>`: PR review (`git diff <branch>...HEAD`)
   - `--commit <sha>`: specific commit (`git diff <sha>~1..<sha>`)
3. If no changes → "No changes to review." Stop.
4. Show file list and line count to user.
5. Show effective command, then execute:

**Codex (default):**
```bash
codex review 2>&1                                     # uncommitted
codex review --base main 2>&1                         # PR review
codex review --commit abc123 2>&1                     # specific commit
codex review "focus on auth boundaries" 2>&1          # custom instructions
codex review -c model_reasoning_effort=high 2>&1      # override reasoning
```
NOTE: `codex review` does NOT support `-p` (profile). Use `-c` for overrides.

**Gemini (--via gemini):**
```bash
# Check diff size first
DIFF_LINES=$(git diff | wc -l)
if [ "$DIFF_LINES" -eq 0 ]; then echo "No changes to review."; exit 0; fi

ERR=$(mktemp)

# Small diff (<300 lines) — single call
RESP=$(git diff | gemini -p "You are a code reviewer. Review the diff on stdin. \
Report ONLY: bugs, security issues, performance problems. Skip style. \
Format: list each finding as '- [file:line] severity: description'" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — exit 1 on failure


# Large diff (>300 lines) — MUST chunk by file
for FILE in $(git diff --name-only); do
  RESP=$(git diff -- "$FILE" | gemini -p "Review this diff for $FILE. Report bugs, security, performance only." -o text 2>/dev/null)
  # collect per-file findings
done
```

For `--base main`:
```bash
git diff main...HEAD | gemini -p "<review prompt>" -o text 2>"$ERR"
```

6. Parse findings into categories:
   - **Bugs / logic errors** — always report
   - **Security issues** — always report
   - **Performance concerns** — report if non-trivial
   - **Style / formatting** — IGNORE (different model aesthetics)
7. Present to user in structured format:
```
## Review Findings (<provider>)

### Bugs
- [file:line] description

### Security
- [file:line] description

### Performance
- [file:line] description

### Summary
X issues found. Style issues filtered out.
```
8. Ask user which findings to address. Never auto-apply fixes.

---

### ask

**Purpose:** Get an external AI's opinion for debate or second perspective.

**Flow:**
1. Extract the question from user input.
2. Show effective command, then execute:

**Codex (default):**
```bash
OUT=$(mktemp) || { echo "Cannot create temp file"; return 1; }
codex exec --ephemeral -C "$PWD" -p "${PROFILE:-fast}" \
  --output-last-message "$OUT" "<prompt>" 2>&1
ANSWER=$(cat "$OUT"); rm -f "$OUT"
```

**Codex with `--with-diff`:**
```bash
OUT=$(mktemp)
{ echo "Context (current changes):"; git diff; echo ""; echo "Question: <prompt>"; } | \
  codex exec --ephemeral -C "$PWD" -p fast \
  --output-last-message "$OUT" - 2>&1
ANSWER=$(cat "$OUT"); rm -f "$OUT"
```

**Gemini (--via gemini):**
```bash
ERR=$(mktemp)
RESP=$(gemini -p "<prompt>" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — return 1 on failure
```

**Gemini with `--with-diff`:**
```bash
ERR=$(mktemp)
RESP=$(git diff | gemini -p "<prompt>" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — return 1 on failure
```

3. Present BOTH perspectives:
```
## Two Perspectives

### <Provider> (<model>)
[Provider's answer, summarized]

### Claude — Own Analysis
[Your own independent analysis of the same question]

### Key Differences
[Where the two perspectives diverge]

### Recommendation
[Your synthesis — what you'd recommend and why]
```
4. Let user decide. Do not auto-implement either perspective.

---

### implement

**Purpose:** Delegate implementation to an external AI in an isolated worktree.

**Flow:**
1. Extract the implementation spec from user input.
2. Verify we're in a git repository. If not → stop.
3. Create an isolated worktree:
```bash
BRANCH="arbiter/$(date +%s)"
git worktree add "/tmp/$BRANCH" -b "$BRANCH" HEAD
```
4. Show effective command, then execute:

**Codex (default):**
```bash
codex exec --full-auto -C "/tmp/$BRANCH" "<spec>" 2>&1
```

**Gemini (--via gemini):**
```bash
# IMPORTANT: subshell to avoid changing Claude's CWD
(cd "/tmp/$BRANCH" && gemini -p "<spec>" -y -o text 2>/dev/null)
```

5. Capture output and check exit code.
6. Show the diff:
```bash
git -C "/tmp/$BRANCH" diff HEAD
```
7. Present results:
```
## Implementation (<provider>)

### Files Changed
[list of files with +/- lines]

### Diff
[key changes, summarized — full diff available on request]

### Claude's Assessment
[Your review: correctness, style, completeness]

### Options
1. Merge into current branch
2. Cherry-pick specific files
3. Discard
```
8. If merge:
```bash
git -C "/tmp/$BRANCH" add -A && git -C "/tmp/$BRANCH" commit -m "arbiter: <spec summary>"
git merge "$BRANCH"
git worktree remove "/tmp/$BRANCH"
```
9. If discard:
```bash
git worktree remove --force "/tmp/$BRANCH"
git branch -D "$BRANCH"
```
10. On ANY error, always clean up worktree. Remind user: `git worktree list | grep arbiter/` for manual cleanup.

---

### panel

**Purpose:** Ask BOTH providers the same question, compare answers.

**IMPORTANT: Run codex and gemini in TRUE PARALLEL using `run_in_background: true`.** Normal parallel Bash tool calls are serialized by Claude Code. Background execution is the only way to get real concurrency.

**Flow:**
1. Extract the question.
2. Launch BOTH providers as background tasks — two Bash tool calls with `run_in_background: true`:

**Bash call 1 (codex) — run_in_background: true:**
```bash
OUT=$(mktemp)
codex exec --ephemeral -C "$PWD" -p fast --output-last-message "$OUT" "<prompt>" 2>&1
echo "---CODEX_ANSWER---"
cat "$OUT"; rm -f "$OUT"
```

**Bash call 2 (gemini) — run_in_background: true:**
```bash
ERR=$(mktemp)
RESP=$(gemini -p "<prompt>" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — report GEMINI_ERROR on failure
echo "---GEMINI_ANSWER---"
echo "$RESP"
```

3. Collect both results using `TaskOutput` tool (block: true) for each background task ID. Form your own independent analysis.
4. Present with voting structure:
```
## Panel: "<question>"

### Codex
[response summary]

### Gemini
[response summary]

### Claude — Own Analysis
[Claude's independent analysis]

### Consensus (N/3 agree)
[What all or majority agree on]

### Split Decision
[Where they diverge — with Claude's recommendation]
```

**Partial failure OK** — if one provider fails, report failure and show the others.

---

### quorum

**Purpose:** Formal go/no-go gate for high-stakes decisions. Returns APPROVE/BLOCK/NEEDS_INFO.

**When to use:** Architecture choices, schema migrations, destructive operations, security decisions. For open-ended questions, use `panel` instead.

Quorum always runs both providers. No `--via` flag.

#### Round 1 — Independent Assessment (Codex + Gemini, parallel)

1. Build context:
   - Always: the question/proposal text
   - If `--with-diff`: prepend `git diff` output
   - State snapshot: `git status --short` + `git log --oneline -3` (equalize context across providers)
2. Contract prompt sent to BOTH providers:
   ```
   You are reviewing a proposal for a high-stakes decision.
   Respond in EXACTLY this format (no other text):

   VERDICT: APPROVE | BLOCK | NEEDS_INFO
   BLOCKING_ISSUES: (numbered list or "none")
   ASSUMPTIONS: (numbered list)
   EVIDENCE_NEEDED: (what facts would change your verdict)
   RATIONALE: (2-3 sentences)

   Rules:
   - BLOCK if: data loss risk, security vulnerability, irreversible without rollback
   - NEEDS_INFO if: ambiguous, missing context, cannot evaluate
   - APPROVE if: sound, safe, reversible or has rollback plan
   ```
3. Launch Codex + Gemini with `run_in_background: true` (Bash tool `timeout: 120000`) — same patterns as panel mode:

**Bash call 1 (codex) — run_in_background: true:**
```bash
OUT=$(mktemp) || { echo "CODEX_ERROR: cannot create temp file"; exit 1; }
CONTEXT=""
# If --with-diff: CONTEXT="Context (current changes):\n$(git diff)\n\n"
STATE="State:\n$(git status --short)\n$(git log --oneline -3)\n\n"
PROMPT="${CONTEXT}${STATE}<contract_prompt>\n\n---\n\n<question>"
echo "$PROMPT" | codex exec --ephemeral -C "$PWD" -p fast \
  --output-last-message "$OUT" - 2>&1
echo "---CODEX_ANSWER---"
cat "$OUT"; rm -f "$OUT"
```

**Bash call 2 (gemini) — run_in_background: true:**
```bash
CONTEXT=""
# If --with-diff: CONTEXT="Context (current changes):\n$(git diff)\n\n"
STATE="State:\n$(git status --short)\n$(git log --oneline -3)\n\n"
PROMPT="${CONTEXT}${STATE}<contract_prompt>\n\n---\n\n<question>"
ERR=$(mktemp)
RESP=$(gemini -p "$PROMPT" -o text 2>"$ERR")
EXIT=$?
# Standard Gemini error handling (see Error Handling §7)
if [ $EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials" "$ERR"; then echo "GEMINI_ERROR: auth"
  else echo "GEMINI_ERROR: exit $EXIT"; cat "$ERR"
  fi
fi
rm -f "$ERR"
echo "---GEMINI_ANSWER---"
echo "$RESP"
```

4. Collect both results using `TaskOutput` tool (block: true) for each background task ID.

#### Parse Validation

- If provider response doesn't contain `VERDICT:` line → mark as `PARSE_ERROR` (not APPROVE, not N/A)
- If Bash tool timed out → mark as `TIMEOUT`
- Minimum 1 valid external response required. If both failed → "Quorum unavailable — fewer than 1 external provider responded. Re-run or use /arbiter ask."

#### Round 2 — Claude Synthesis

Claude sees both responses and applies deterministic policy FIRST, then adds analysis.

**Decision policy (deterministic, applied before LLM synthesis):**
1. Both BLOCK → final BLOCK
2. Both APPROVE → final APPROVE
3. Both NEEDS_INFO → final NEEDS_INFO, consolidate questions
4. One BLOCK + one APPROVE → **split decision, Claude = tiebreaker**
5. Any BLOCK + NEEDS_INFO → final BLOCK (conservative)
6. One APPROVE + NEEDS_INFO → final NEEDS_INFO
7. One valid + one failed → present single result with degraded quorum warning

**For split decisions (case 4) — Claude as tiebreaker:**
Use adversarial prompt: "The providers disagree. Your job: first, state the strongest case FOR the BLOCK. Then state the strongest case AGAINST it. Only then give your tiebreaker verdict with rationale."
This mitigates confirmation bias by forcing devil's advocate.

#### Output Format

```
## Quorum: "<question>"

### Votes

| Provider | Verdict | Key Issue |
|----------|---------|-----------|
| Codex    | BLOCK   | SQL injection in query builder |
| Gemini   | APPROVE | — |

### Analysis

[Claude's synthesis: why they disagree, strength of the BLOCK argument, missed risks]

### Verdict: BLOCKED | APPROVED | NEEDS_INFO
[Rationale based on deterministic policy + Claude analysis]

### Action Items
[If BLOCK: what to fix. If NEEDS_INFO: what to answer first]
```

#### Logging

Append to `~/vicc/state/quorum-log.md` conditionally (if dir exists):
```bash
if [ -d ~/vicc/state ]; then
cat >> ~/vicc/state/quorum-log.md << EOF

## <timestamp>

**Question:** <first 3 lines>
**Result:** BLOCKED (1 BLOCK, 1 APPROVE — tiebreaker: Claude → BLOCK)
**Votes:** Codex=BLOCK Gemini=APPROVE
**Source:** arbiter quorum
EOF
fi
```

---

### verify

**Purpose:** Cross-model verification with claim decomposition. Catches hallucinations, agreement bias, and unsupported claims that panel/quorum miss.

**When to use:** Factual questions, technical claims, "is X true?", any answer where correctness matters more than speed.

**verify always runs both providers. No `--via` flag.**

#### Round 1 — Independent Generation (parallel)

Same pattern as panel: two Bash calls with `run_in_background: true`.
Each provider answers the question independently, with NO shared context.

**Bash call 1 (codex) — run_in_background: true:**
```bash
OUT=$(mktemp) || { echo "CODEX_ERROR: cannot create temp file"; exit 1; }
CONTEXT=""
# If --with-diff: CONTEXT="Context (current changes):\n$(git diff)\n\n"
codex exec --ephemeral -C "$PWD" -p fast \
  --output-last-message "$OUT" "<prompt>" 2>&1
echo "---CODEX_ANSWER---"
cat "$OUT"; rm -f "$OUT"
```

**Bash call 2 (gemini) — run_in_background: true:**
```bash
CONTEXT=""
# If --with-diff: CONTEXT="Context (current changes):\n$(git diff)\n\n"
ERR=$(mktemp)
RESP=$(gemini -p "${CONTEXT}<prompt>" -o text 2>"$ERR")
EXIT=$?
# Standard Gemini error handling (see Error Handling §7)
if [ $EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials" "$ERR"; then echo "GEMINI_ERROR: auth"
  else echo "GEMINI_ERROR: exit $EXIT"; cat "$ERR"
  fi
fi
rm -f "$ERR"
echo "---GEMINI_ANSWER---"
echo "$RESP"
```

Collect both results using `TaskOutput` tool (block: true) for each background task ID.

#### Round 2 — Claim Decomposition & Comparison (Claude)

Claude performs (NO external calls — this is Claude's own analysis):

1. Decompose BOTH answers into numbered atomic claims
   - One verifiable fact per claim
   - Skip opinions, hedging, tautologies
   - Max 15 claims per answer

2. Compare claims between answers:
   - **AGREED**: both answers make the same claim
   - **CONTESTED**: one answer claims X, other claims not-X or different
   - **UNIQUE**: claim appears in only one answer (not contradicted)

3. If zero CONTESTED claims → skip Round 3, go to synthesis

#### Round 3 — Adversarial Cross-Verification (parallel, conditional)

Only runs if Round 2 found CONTESTED claims.

Build adversarial verification prompt with ANONYMOUS contested claims
(Claude strips which model said what):

```
You are a skeptical fact-checker. Your goal is to FIND FLAWS, not confirm.

For each claim below, respond:
- PASS: claim is correct (explain why with supporting evidence)
- FAIL: claim is wrong (provide specific counterexample or contradiction)
- UNKNOWN: cannot determine (explain what evidence is needed)

Do NOT agree by default. If you cannot verify, say UNKNOWN.

Claims:
1. [anonymous claim]
2. [anonymous claim]
...
```

Two Bash calls with `run_in_background: true`:

**Bash call 1 (codex) — run_in_background: true:**
```bash
OUT=$(mktemp) || { echo "CODEX_ERROR: cannot create temp file"; exit 1; }
echo "$ADVERSARIAL_PROMPT" | codex exec --ephemeral -C "$PWD" -p fast \
  --output-last-message "$OUT" - 2>&1
echo "---CODEX_VERIFY---"
cat "$OUT"; rm -f "$OUT"
```

**Bash call 2 (gemini) — run_in_background: true:**
```bash
ERR=$(mktemp)
RESP=$(gemini -p "$ADVERSARIAL_PROMPT" -o text 2>"$ERR")
EXIT=$?
if [ $EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials" "$ERR"; then echo "GEMINI_ERROR: auth"
  else echo "GEMINI_ERROR: exit $EXIT"; cat "$ERR"
  fi
fi
rm -f "$ERR"
echo "---GEMINI_VERIFY---"
echo "$RESP"
```

Collect both results using `TaskOutput` tool (block: true) for each background task ID.

#### Synthesis

For each claim, determine status:
- **VERIFIED**: agreed by both in Round 1, OR both verifiers PASS in Round 3
- **CONTESTED**: verifiers disagree (one PASS, one FAIL) → surface both arguments
- **REJECTED**: both verifiers FAIL → flag explicitly with counterexamples
- **UNCERTAIN**: both verifiers UNKNOWN → may need external evidence

#### Output Format

```
## Verify: "<question>"

### Answer (synthesized)
[Merged answer using only VERIFIED claims. Contested claims presented as "X, though this is disputed because Y"]

### Claim Analysis

| # | Claim | Status | Detail |
|---|-------|--------|--------|
| 1 | ... | VERIFIED | Both providers agree |
| 2 | ... | CONTESTED | Codex: PASS, Gemini: FAIL — [counterexample] |
| 3 | ... | REJECTED | Both found errors — [correction] |

### Contested Claims (detail)
[For each CONTESTED claim: both sides of the argument]

### Confidence
X/Y claims verified, Z contested, W rejected.
[HIGH / MEDIUM / LOW / NEEDS HUMAN REVIEW]

Thresholds:
- HIGH: >80% VERIFIED, 0 REJECTED
- MEDIUM: >60% VERIFIED, ≤1 REJECTED
- LOW: <60% VERIFIED or >1 REJECTED
- NEEDS HUMAN REVIEW: >30% CONTESTED+REJECTED
```

#### Logging

Same pattern as quorum — append to `~/vicc/state/quorum-log.md` conditionally (if dir exists):
```bash
if [ -d ~/vicc/state ]; then
cat >> ~/vicc/state/quorum-log.md << EOF

## <timestamp>

**Question:** <first 3 lines>
**Result:** <confidence> (X verified, Y contested, Z rejected)
**Rounds:** <2 or 3>
**Source:** arbiter verify
EOF
fi
```

---

### continue

**Purpose:** Resume the last Gemini session with a follow-up.

Codex `continue` is NOT supported in v1 (`codex exec resume --last` lacks `--output-last-message`).

**Flow:**
1. If `--via codex` → error: "Continue not supported for codex in v1. Use --via gemini."
2. Extract the follow-up prompt.
3. Execute:
```bash
ERR=$(mktemp)
RESP=$(gemini -r latest -p "<follow-up>" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — return 1 on failure
```
Note: Gemini sessions are CWD-scoped.

4. Present the response and your own commentary.

## Help

When invoked without arguments (`/arbiter`), show:

```
Arbiter — multi-AI orchestrator

Providers:
  codex   <status>  (<version>)
  gemini  <status>  (<version>)

Modes:
  review      Code review (default: codex)
  ask         Ask a question (default: codex)
  implement   Delegate implementation (default: codex)
  panel       Ask all providers, compare
  quorum      Formal go/no-go verdict (high-stakes only)
  verify      Cross-verification with claim decomposition
  continue    Resume last session (gemini only)

Usage: /arbiter <mode> [--via codex|gemini] [--base <branch>] [args]

Examples:
  /arbiter review
  /arbiter review --base main
  /arbiter ask "should I use async here?"
  /arbiter ask --via gemini --with-diff "is this thread-safe?"
  /arbiter panel "best testing strategy for this module?"
  /arbiter implement --via gemini "add input validation"
  /arbiter quorum "should I drop the users table and recreate?"
  /arbiter quorum --with-diff "are these migration changes safe?"
  /arbiter verify "Is SQLite safe for concurrent writes in production?"
  /arbiter verify --with-diff "is this thread-safe?"
```

Provider status: run `which codex && codex --version` / `which gemini && gemini --version`. Show checkmark if found, cross if missing.

## Output Presentation

Before every provider call, show the effective command:
```
> codex review --base main
```

## Error Handling

1. **Binary not found** → "<provider> not found in PATH. Install: <url>"
   - codex: https://github.com/openai/codex
   - gemini: https://github.com/google-gemini/gemini-cli
2. **Auth error** → parse stderr for auth keywords (auth, login, 403, 401, credentials). Message: "Run `codex login` / `gemini login`"
3. **Timeout** → 120s per provider. Kill and report partial output with `[TIMEOUT]` marker.
4. **No changes for review** → check `git diff --stat` first, stop early.
5. **Large diff** → if >300 lines, MUST chunk by file for gemini (codex handles natively).
6. **Not in git repo** → check before review/implement modes, stop with message.
7. **Gemini stderr noise** → always capture stderr to tempfile (`ERR=$(mktemp)`), check for auth keywords, discard noise. Standard pattern:
   ```bash
   ERR=$(mktemp)
   RESP=$(... 2>"$ERR")
   EXIT=$?
   if [ $EXIT -ne 0 ]; then
     if grep -qi "auth\|login\|403\|401\|credentials" "$ERR"; then
       echo "Gemini auth error. Run: gemini login"
     else
       echo "Gemini error (exit $EXIT):"; cat "$ERR"
     fi
     rm -f "$ERR"; return 1
   fi
   rm -f "$ERR"
   ```
   All Gemini calls in every mode MUST use this pattern. Adapt `return 1` / `exit 1` to context.
8. **Orphan worktrees** → on error during implement, always clean up. Note: `git worktree list | grep arbiter/` for manual cleanup.
9. **Quorum parse failure** → if provider response lacks `VERDICT:` line, mark as PARSE_ERROR. Do not count as a vote.
10. **Quorum degraded** → if only 1 of 2 providers responded, warn user: "Degraded quorum (1/2 providers)."

## Backward Compatibility

If triggered via `/codex` (old trigger), handle normally — arbiter processes the command. No redirect message needed, just run the requested mode with codex as the provider.

## Rules

1. **Never auto-apply findings.** Always present to user first.
2. **Filter style disagreements** across all providers — different models have different aesthetics.
3. **Show effective command** before execution for transparency.
4. **Cost awareness** — mention if task seems trivial for the selected provider/model.
5. **Worktree cleanup** on error/discard (implement mode). Never leave orphans.
6. **No anchored review** — never show provider A's full answer to provider B for confirmation. Cross-verification must use anonymous claim decomposition (verify mode) or independent generation (panel/quorum).
7. **Panel: partial failure OK** — if one provider fails, report failure + show others.
8. **Codex review = text** — always parse as plain text, never expect JSON.
9. **Gemini stderr = tempfile** — never `2>/dev/null` blindly, always check for auth errors first.
10. **Quorum: Claude does NOT vote** — only Codex and Gemini vote. Claude moderates and breaks ties.
11. **Quorum: deterministic policy first** — apply decision rules before LLM synthesis. Never override a clear BLOCK without evidence.
12. **Quorum: minimum 1 external** — if both providers fail, abort with error. Never produce a quorum from 0 external votes.
