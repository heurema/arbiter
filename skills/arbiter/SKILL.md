---
name: arbiter
description: >
  Use when user says "/arbiter", "/codex", "ask codex", "ask gemini",
  "codex review", "gemini review", "panel mode", "compare AIs",
  "second opinion from codex", "second opinion from gemini",
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
/arbiter continue "follow-up"               → gemini only (codex not supported in v1)
/arbiter                                     → show help + provider status
```

## Input Parsing

Parse the user's `/arbiter` invocation by matching these patterns in order:

1. Extract mode: first word after `/arbiter` → `review|ask|implement|panel|continue`
2. Extract `--via <provider>`: if present, set provider (codex|gemini)
3. Extract mode-specific flags:
   - review: `--base <branch>`, `--commit <sha>`
   - ask: `--with-diff`
4. Remaining quoted string → prompt/question/spec
5. If no mode → show help

**Conflict rules:**
- `--base` + `--commit` → error: "Choose one: --base or --commit"
- `--via unknown` → error: "Unknown provider. Available: codex, gemini"
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
EXIT=$?
if [ $EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials" "$ERR"; then
    echo "Gemini auth error. Run: gemini login"
  else
    echo "Gemini error (exit $EXIT):"; cat "$ERR"
  fi
  rm -f "$ERR"; exit 1
fi
rm -f "$ERR"

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

**Gemini with `--with-diff`:**
```bash
ERR=$(mktemp)
RESP=$(git diff | gemini -p "<prompt>" -o text 2>"$ERR")
# Same error handling
rm -f "$ERR"
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

Runs providers **sequentially** (not parallel — Claude Code executes one Bash at a time).

**Flow:**
1. Extract the question.
2. Run codex:
```bash
OUT=$(mktemp)
codex exec --ephemeral -C "$PWD" -p fast --output-last-message "$OUT" "<prompt>" 2>&1
CODEX_ANSWER=$(cat "$OUT"); rm -f "$OUT"
```
3. Run gemini:
```bash
ERR=$(mktemp)
GEMINI_ANSWER=$(gemini -p "<prompt>" -o text 2>"$ERR")
# error handling...
rm -f "$ERR"
```
4. Form your own independent analysis.
5. Present with voting structure:
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
  continue    Resume last session (gemini only)

Usage: /arbiter <mode> [--via codex|gemini] [--base <branch>] [args]

Examples:
  /arbiter review
  /arbiter review --base main
  /arbiter ask "should I use async here?"
  /arbiter ask --via gemini --with-diff "is this thread-safe?"
  /arbiter panel "best testing strategy for this module?"
  /arbiter implement --via gemini "add input validation"
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
7. **Gemini stderr noise** → always capture to tempfile, parse for errors, discard noise.
8. **Orphan worktrees** → on error during implement, always clean up. Note: `git worktree list | grep arbiter/` for manual cleanup.

## Backward Compatibility

If triggered via `/codex` (old trigger), handle normally — arbiter processes the command. No redirect message needed, just run the requested mode with codex as the provider.

## Rules

1. **Never auto-apply findings.** Always present to user first.
2. **Filter style disagreements** across all providers — different models have different aesthetics.
3. **Show effective command** before execution for transparency.
4. **Cost awareness** — mention if task seems trivial for the selected provider/model.
5. **Worktree cleanup** on error/discard (implement mode). Never leave orphans.
6. **No recursive AI review** — don't use provider A to review provider B's output.
7. **Panel: partial failure OK** — if one provider fails, report failure + show others.
8. **Codex review = text** — always parse as plain text, never expect JSON.
9. **Gemini stderr = tempfile** — never `2>/dev/null` blindly, always check for auth errors first.
