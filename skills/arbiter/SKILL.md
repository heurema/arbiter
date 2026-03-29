---

name: arbiter
description: Multi-AI orchestrator — dispatch to Codex CLI and Gemini CLI for review, ask, implement, panel, quorum, continue, diverge, and doctor modes. Diverge generates 3 independent implementations in isolated worktrees with different strategy hints, then compares and selects the best solution. Doctor checks provider health and reports partial success when some providers are unavailable.
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
| ask | `codex exec --ephemeral -C "$PWD" $CODEX_PROFILE_FLAG --output-last-message "$OUT" "<prompt>"` | `gemini -p "<prompt>" -o text` | codex |
| review | `codex review [flags]` | `git diff \| gemini -p "<review_prompt>" -o text` | codex |
| implement | `codex exec --full-auto -C "<worktree>" "<spec>"` | `(cd "<worktree>" && gemini -p "<spec>" -y -o text)` | codex |
| continue | Not supported in v1 | `gemini -r latest -p "<follow-up>" -o text` | gemini |
| diverge | 3 worktrees + codex/gemini | 3 worktrees + codex/gemini | all |
| doctor | N/A | N/A | N/A |

**Note:** `quorum` and `verify` are not in this table — they are multi-provider protocols, not single-provider dispatch. Quorum always runs both providers (see [quorum mode](#quorum)). Verify always runs both providers (see [verify mode](#verify)).

### Model Resolution

Before provider CLI calls, resolve model from config if available:

```bash
_PROVIDERS_CONFIG="${EMPORIUM_PROVIDERS_CONFIG:-$HOME/.claude/emporium-providers.local.md}"
_resolve_model() {
  local task="$1" provider="$2"
  [ -r "$_PROVIDERS_CONFIG" ] || { echo ""; return 0; }
  python3 -c "
import re, sys, os
config_path = os.environ.get('EMPORIUM_PROVIDERS_CONFIG', os.path.expanduser('~/.claude/emporium-providers.local.md'))
try:
    text = open(config_path).read()
except Exception:
    sys.exit(0)
if text.startswith('\ufeff'):
    text = text[1:]
m = re.match(r'^---\s*\n(.*?)\n---', text, re.DOTALL)
if not m:
    sys.exit(0)
def parse_yaml(yaml_text):
    result = {}
    stack = [(-1, result)]
    for line in yaml_text.splitlines():
        if not line.strip() or line.strip().startswith('#'):
            continue
        indent = len(line) - len(line.lstrip())
        kv = line.strip()
        while len(stack) > 1 and indent <= stack[-1][0]:
            stack.pop()
        if ':' not in kv:
            if kv.startswith('- '):
                v = kv[2:].strip().strip('\"')
                d = stack[-1][1]
                for k in reversed(list(d.keys())):
                    if isinstance(d[k], list):
                        d[k].append(v)
                        break
            continue
        key, _, value = kv.partition(':')
        key = key.strip()
        value = value.strip().strip('\"')
        d = stack[-1][1]
        if not value:
            nd = {}
            d[key] = nd
            stack.append((indent, nd))
        else:
            try: d[key] = int(value)
            except ValueError:
                if value.lower() in ('true','false'): d[key] = value.lower() == 'true'
                else: d[key] = value
    return result
try:
    import yaml
    cfg = yaml.safe_load(m.group(1)) or {}
except ImportError:
    cfg = parse_yaml(m.group(1))
task = sys.argv[1]
provider = sys.argv[2]
route = cfg.get('routing', {})
model = ''
if task in route and isinstance(route[task], dict) and provider in route[task]:
    model = str(route[task][provider])
elif 'default' in route and isinstance(route['default'], dict) and provider in route['default']:
    model = str(route['default'][provider])
else:
    pd = cfg.get('defaults', {}).get(provider, {})
    if isinstance(pd, dict): model = str(pd.get('model', ''))
if model and not re.match(r'^[A-Za-z0-9._:-]+$', model): model = ''
print(model, end='')
" "$task" "$provider" 2>/dev/null
}
```

**Usage pattern:**
```bash
MODEL=$(_resolve_model "<task>" "<provider>")
MODEL_FLAG=""
[ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
```

### Codex Profile Resolution

Codex `-p` (profile) is optional. If `ARBITER_CODEX_PROFILE` is set, it is passed to all `codex exec` calls. If unset, codex uses its default profile.

```bash
CODEX_PROFILE_FLAG=""
[ -n "${ARBITER_CODEX_PROFILE:-}" ] && CODEX_PROFILE_FLAG="-p $ARBITER_CODEX_PROFILE"
```

NOTE: This version passes task/provider as sys.argv instead of bash string interpolation (addresses the security concern from signum code review).

For Gemini CLI, verified headless model strings in this environment are `auto-gemini-3`, `gemini-3.1-pro-preview`, `gemini-3-flash-preview`, `gemini-2.5-pro`, `gemini-2.5-flash`, `pro`, and `flash`. If Gemini 3 preview is not enabled for the account, fall back to `pro` / `flash` or `gemini-2.5-*`.

### Capability Notes
- **Claude Code CLI** supports `--model` with aliases like `haiku`, `sonnet`, and `opus`. For nested Claude CLI calls, always sanitize the environment first: `env -u ANTHROPIC_API_KEY -u CLAUDECODE claude ...` so subscription auth is not shadowed by an unrelated API-key env var.
- **Codex** uses `-p` for profiles (fast, default, full). Profile is optional — set `ARBITER_CODEX_PROFILE` env var to enable. `-c model_reasoning_effort=high` for reasoning override.
- **Gemini** uses `-o text` always (not `-o json` — that wraps in transport envelope). `-y` for auto-accept in implement.
- **Gemini** model IDs should be passed via `--model`. For Gemini 3 routing, prefer verified headless strings like `auto-gemini-3`, `gemini-3.1-pro-preview`, and `gemini-3-flash-preview`. Use `pro` / `flash` or `gemini-2.5-*` as fallback when preview access is unavailable.
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
/arbiter diverge "spec"                        → 3 providers, default strategies
/arbiter diverge --lite "spec"                  → design-level only, no code
/arbiter diverge --strategies "a,b,c" "spec"    → custom strategy hints
/arbiter diverge --providers "claude,codex"      → specific providers
/arbiter diverge --timeout 600 "spec"           → custom timeout (default: 300s)
/arbiter diverge --run-tests "spec"             → test all 3 before evaluation
/arbiter doctor                              → health check (binary + smoke API)
```

## Input Parsing

Parse the user's `/arbiter` invocation by matching these patterns in order:

1. Extract mode: first word after `/arbiter` → `review|ask|implement|panel|quorum|verify|continue|diverge|doctor`
2. Extract `--via <provider>`: if present, set provider (codex|gemini). Not applicable to `quorum` (always both).
3. Extract mode-specific flags:
   - review: `--base <branch>`, `--commit <sha>`
   - ask: `--with-diff`
   - quorum: `--with-diff`
   - verify: `--with-diff`
   - diverge: `--lite`, `--strategies <csv>`, `--providers <csv>`, `--timeout <seconds>`, `--run-tests`
4. Remaining quoted string → prompt/question/spec
5. If no mode → show help

**Conflict rules:**
- `--base` + `--commit` → error: "Choose one: --base or --commit"
- `--via unknown` → error: "Unknown provider. Available: codex, gemini"
- `quorum --via <any>` → error: "Quorum always runs both providers. No --via flag."
- `verify --via <any>` → error: "Verify always runs both providers. No --via flag."
- `continue --via codex` → error: "Continue not supported for codex in v1. Use --via gemini."
- `diverge --providers` with only 1 provider → warn: "Only 1 provider. Results will show 3 strategies from 1 model."
- `diverge --lite --run-tests` → error: "Cannot run tests in lite mode (no code)."

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
# NOTE: codex review may not support --model. If unsupported, omit MODEL_FLAG.
MODEL=$(_resolve_model "review" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
codex review $MODEL_FLAG 2>&1                                     # uncommitted
codex review $MODEL_FLAG --base main 2>&1                         # PR review
codex review $MODEL_FLAG --commit abc123 2>&1                     # specific commit
codex review $MODEL_FLAG "focus on auth boundaries" 2>&1          # custom instructions
codex review $MODEL_FLAG -c model_reasoning_effort=high 2>&1      # override reasoning
```
NOTE: `codex review` does NOT support `-p` (profile). Use `-c` for overrides. `--model` support is unconfirmed — test with `codex review --help` and remove MODEL_FLAG if unsupported.

**Gemini (--via gemini):**
```bash
# Determine diff source — include both unstaged and staged changes
# Validate refs before use to prevent bad-ref failures showing as empty diffs
if [ -n "$BASE" ]; then
  git rev-parse --verify "$BASE" >/dev/null 2>&1 || { echo "Error: invalid base ref '$BASE'"; exit 1; }
  DIFF_ARGS=("${BASE}...HEAD")
elif [ -n "$COMMIT" ]; then
  git rev-parse --verify "$COMMIT" >/dev/null 2>&1 || { echo "Error: invalid commit '$COMMIT'"; exit 1; }
  DIFF_ARGS=("${COMMIT}~1..${COMMIT}")
else
  # Default: unstaged + staged combined
  DIFF_ARGS=("HEAD")
fi

DIFF_LINES=$(git diff "${DIFF_ARGS[@]}" | wc -l)
if [ "$DIFF_LINES" -eq 0 ]; then echo "No changes to review."; exit 0; fi

MODEL=$(_resolve_model "review" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
ERR=$(mktemp)

# Small diff (<300 lines) — single call
RESP=$(git diff "${DIFF_ARGS[@]}" | gemini $MODEL_FLAG -p "You are a code reviewer. Review the diff on stdin. \
Report ONLY: bugs, security issues, performance problems. Skip style. \
Format: list each finding as '- [file:line] severity: description'" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — exit 1 on failure


# Large diff (>300 lines) — MUST chunk by file
ERR_CHUNK=$(mktemp)
git diff "${DIFF_ARGS[@]}" --name-only -z | while IFS= read -r -d '' FILE; do
  RESP=$(git diff "${DIFF_ARGS[@]}" -- "$FILE" | gemini $MODEL_FLAG -p "Review this diff for $FILE. Report bugs, security, performance only." -o text 2>"$ERR_CHUNK")
  # Standard Gemini error handling per chunk
  # collect per-file findings
done
rm -f "$ERR_CHUNK"
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
MODEL=$(_resolve_model "ask" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
OUT=$(mktemp) || { echo "Cannot create temp file"; return 1; }
codex exec --ephemeral -C "$PWD" $CODEX_PROFILE_FLAG $MODEL_FLAG \
  --output-last-message "$OUT" "<prompt>" 2>&1
ANSWER=$(cat "$OUT"); rm -f "$OUT"
```

**Codex with `--with-diff`:**
```bash
MODEL=$(_resolve_model "ask" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
OUT=$(mktemp)
{ echo "Context (current changes):"; git diff; echo ""; echo "Question: <prompt>"; } | \
  codex exec --ephemeral -C "$PWD" $CODEX_PROFILE_FLAG $MODEL_FLAG \
  --output-last-message "$OUT" - 2>&1
ANSWER=$(cat "$OUT"); rm -f "$OUT"
```

**Gemini (--via gemini):**
```bash
MODEL=$(_resolve_model "ask" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
ERR=$(mktemp)
RESP=$(gemini $MODEL_FLAG -p "<prompt>" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — return 1 on failure
```

**Gemini with `--with-diff`:**
```bash
MODEL=$(_resolve_model "ask" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
ERR=$(mktemp)
RESP=$(git diff | gemini $MODEL_FLAG -p "<prompt>" -o text 2>"$ERR")
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
BRANCH="arbiter-$(date +%s)-$$"
WORKTREE=$(mktemp -d "/tmp/arbiter-XXXXXX")
git worktree add "$WORKTREE" -b "$BRANCH" HEAD
```
4. Show effective command, then execute:

**Codex (default):**
```bash
MODEL=$(_resolve_model "implement" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
codex exec --full-auto -C "$WORKTREE" $MODEL_FLAG "<spec>" 2>&1
```

**Gemini (--via gemini):**
```bash
MODEL=$(_resolve_model "implement" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
ERR=$(mktemp)
# IMPORTANT: subshell to avoid changing Claude's CWD
(cd "$WORKTREE" && gemini $MODEL_FLAG -p "<spec>" -y -o text 2>"$ERR")
GEMINI_EXIT=$?
if [ $GEMINI_EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials" "$ERR"; then
    echo "GEMINI_ERROR: auth — run: gemini login"
  else
    echo "GEMINI_ERROR: exit $GEMINI_EXIT"; cat "$ERR"
  fi
fi
rm -f "$ERR"
```

5. Capture output and check exit code.
6. Show the diff:
```bash
git -C "$WORKTREE" diff HEAD
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
git -C "$WORKTREE" add -A && git -C "$WORKTREE" commit -m "arbiter: <spec summary>"
git merge "$BRANCH"
git worktree remove "$WORKTREE"
```
9. If discard:
```bash
git worktree remove --force "$WORKTREE"
git branch -D "$BRANCH"
```
10. On ANY error, always clean up worktree. Remind user: `git worktree list | grep arbiter/` for manual cleanup.

---

### panel

**Purpose:** Ask BOTH providers the same question, compare answers.

**IMPORTANT: Run codex and gemini in TRUE PARALLEL using `run_in_background: true`.** Normal parallel Bash tool calls are serialized by Claude Code. Background execution is the only way to get real concurrency.

**Flow:**
1. Extract the question.
1a. After collecting all provider results, compute and show Run Summary (see [Run Summary](#run-summary)). On FAILURE → stop with "All providers failed. Cannot form panel."
2. Launch BOTH providers as background tasks — two Bash tool calls with `run_in_background: true`:

**Bash call 1 (codex) — run_in_background: true:**
```bash
MODEL=$(_resolve_model "ask" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
OUT=$(mktemp)
codex exec --ephemeral -C "$PWD" $CODEX_PROFILE_FLAG $MODEL_FLAG --output-last-message "$OUT" "<prompt>" 2>&1
echo "---CODEX_ANSWER---"
cat "$OUT"; rm -f "$OUT"
```

**Bash call 2 (gemini) — run_in_background: true:**
```bash
MODEL=$(_resolve_model "ask" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
ERR=$(mktemp)
RESP=$(gemini $MODEL_FLAG -p "<prompt>" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — report GEMINI_ERROR on failure
echo "---GEMINI_ANSWER---"
echo "$RESP"
```

3. Collect both results using `TaskOutput` tool (block: true) for each background task ID. Form your own independent analysis.
4. Present with voting structure. If a provider's response includes a `CONFIDENCE:` line (e.g., from quorum-style prompts or self-assessed), extract and display it. Otherwise omit the confidence annotation.
```
## Panel: "<question>"

### Codex [Confidence: 0.85]
[response summary]

### Gemini [Confidence: 0.70]
[response summary]

### Claude — Own Analysis
[Claude's independent analysis]

### Consensus (N/3 agree)
[What all or majority agree on]

### Split Decision
[Where they diverge — with Claude's recommendation, noting confidence asymmetry if present]
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
   CONFIDENCE: (0.0–1.0, where 1.0 = absolute certainty in your verdict)
   BLOCKING_ISSUES: (numbered list or "none")
   ASSUMPTIONS: (numbered list)
   EVIDENCE_NEEDED: (what facts would change your verdict)
   RATIONALE: (2-3 sentences)

   Rules:
   - BLOCK if: data loss risk, security vulnerability, irreversible without rollback
   - NEEDS_INFO if: ambiguous, missing context, cannot evaluate
   - APPROVE if: sound, safe, reversible or has rollback plan
   - CONFIDENCE: 0.9+ = very certain, 0.7-0.9 = confident, 0.5-0.7 = uncertain, <0.5 = low confidence
   ```
3. Launch Codex + Gemini with `run_in_background: true` (Bash tool `timeout: 120000`) — same patterns as panel mode:

**Bash call 1 (codex) — run_in_background: true:**
```bash
MODEL=$(_resolve_model "review" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
OUT=$(mktemp) || { echo "CODEX_ERROR: cannot create temp file"; exit 1; }
CONTEXT=""
# If --with-diff: CONTEXT="Context (current changes):\n$(git diff)\n\n"
STATE="State:\n$(git status --short)\n$(git log --oneline -3)\n\n"
PROMPT="${CONTEXT}${STATE}<contract_prompt>\n\n---\n\n<question>"
echo "$PROMPT" | codex exec --ephemeral -C "$PWD" $CODEX_PROFILE_FLAG $MODEL_FLAG \
  --output-last-message "$OUT" - 2>&1
echo "---CODEX_ANSWER---"
cat "$OUT"; rm -f "$OUT"
```

**Bash call 2 (gemini) — run_in_background: true:**
```bash
MODEL=$(_resolve_model "review" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
CONTEXT=""
# If --with-diff: CONTEXT="Context (current changes):\n$(git diff)\n\n"
STATE="State:\n$(git status --short)\n$(git log --oneline -3)\n\n"
PROMPT="${CONTEXT}${STATE}<contract_prompt>\n\n---\n\n<question>"
ERR=$(mktemp)
RESP=$(gemini $MODEL_FLAG -p "$PROMPT" -o text 2>"$ERR")
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

4. Collect both results using `TaskOutput` tool (block: true) for each background task ID. Show Run Summary (see [Run Summary](#run-summary)) before presenting quorum results. PARTIAL_SUCCESS handling subsumes the degraded quorum warning (Error #10).

#### Parse Validation

- If provider response doesn't contain `VERDICT:` line → mark as `PARSE_ERROR` (not APPROVE, not N/A)
- If Bash tool timed out → mark as `TIMEOUT`
- Extract `CONFIDENCE:` value from each response. Parse as float, clamp to [0.0, 1.0]. If `CONFIDENCE:` line is missing or unparseable → default to 0.5 (neutral confidence)
- Minimum 1 valid external response required. If both failed → "Quorum unavailable — fewer than 1 external provider responded. Re-run or use /arbiter ask."

#### Round 2 — Claude Synthesis

Claude sees both responses (including their confidence scores) and applies deterministic policy FIRST, then adds analysis.

**Decision policy (deterministic, applied before LLM synthesis):**
1. Both BLOCK → final BLOCK
2. Both APPROVE → final APPROVE
3. Both NEEDS_INFO → final NEEDS_INFO, consolidate questions
4. One BLOCK + one APPROVE → **confidence-weighted split decision** (see below)
5. Any BLOCK + NEEDS_INFO → final BLOCK (conservative)
6. One APPROVE + NEEDS_INFO → final NEEDS_INFO
7. One valid + one failed → present single result with degraded quorum warning

**For split decisions (case 4) — conservative resolution:**
A BLOCK is a safety signal — confidence weighting must not auto-override it:
- If the BLOCK provider has confidence ≥ 0.8 → final **BLOCK** (high-confidence BLOCK is authoritative)
- Otherwise → **Claude = tiebreaker** with adversarial prompt (BLOCK can only be overridden after explicit adversarial analysis, never by confidence score alone)

When Claude acts as tiebreaker, use adversarial prompt enriched with confidence context: "The providers disagree. Codex: <VERDICT> (confidence: <score>). Gemini: <VERDICT> (confidence: <score>). Your job: first, state the strongest case FOR the BLOCK. Then state the strongest case AGAINST it. Consider that the <higher-confidence provider> is more certain about their assessment. Only then give your tiebreaker verdict with rationale."
This mitigates confirmation bias by forcing devil's advocate while surfacing confidence asymmetry.

#### Output Format

```
## Quorum: "<question>"

### Votes

| Provider | Verdict | Confidence | Key Issue |
|----------|---------|------------|-----------|
| Codex    | BLOCK   | 0.85       | SQL injection in query builder |
| Gemini   | APPROVE | 0.40       | — |

### Analysis

[Claude's synthesis: why they disagree, strength of the BLOCK argument, missed risks. Note confidence asymmetry if significant (≥0.3 gap)]

### Verdict: BLOCKED | APPROVED | NEEDS_INFO
[Rationale based on deterministic policy + confidence weighting + Claude analysis]
[If confidence-weighted resolution applied: "Resolved by confidence weighting: <provider> (<verdict>, confidence <score>) overrides <provider> (<verdict>, confidence <score>)"]

### Action Items
[If BLOCK: what to fix. If NEEDS_INFO: what to answer first]
```

#### Logging

Quorum does not persist any data to disk. Prompt content and verdicts remain in the conversation context only. This is consistent with Arbiter's trust boundary: "No data is stored by Arbiter."

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
MODEL=$(_resolve_model "review" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
OUT=$(mktemp) || { echo "CODEX_ERROR: cannot create temp file"; exit 1; }
CONTEXT=""
# If --with-diff: CONTEXT="Context (current changes):\n$(git diff)\n\n"
codex exec --ephemeral -C "$PWD" $CODEX_PROFILE_FLAG $MODEL_FLAG \
  --output-last-message "$OUT" "<prompt>" 2>&1
echo "---CODEX_ANSWER---"
cat "$OUT"; rm -f "$OUT"
```

**Bash call 2 (gemini) — run_in_background: true:**
```bash
MODEL=$(_resolve_model "review" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
CONTEXT=""
# If --with-diff: CONTEXT="Context (current changes):\n$(git diff)\n\n"
ERR=$(mktemp)
RESP=$(gemini $MODEL_FLAG -p "${CONTEXT}<prompt>" -o text 2>"$ERR")
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

Collect both results using `TaskOutput` tool (block: true) for each background task ID. Show Run Summary (see [Run Summary](#run-summary)) before presenting verify results.

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
MODEL=$(_resolve_model "review" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
OUT=$(mktemp) || { echo "CODEX_ERROR: cannot create temp file"; exit 1; }
echo "$ADVERSARIAL_PROMPT" | codex exec --ephemeral -C "$PWD" $CODEX_PROFILE_FLAG $MODEL_FLAG \
  --output-last-message "$OUT" - 2>&1
echo "---CODEX_VERIFY---"
cat "$OUT"; rm -f "$OUT"
```

**Bash call 2 (gemini) — run_in_background: true:**
```bash
MODEL=$(_resolve_model "review" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
ERR=$(mktemp)
RESP=$(gemini $MODEL_FLAG -p "$ADVERSARIAL_PROMPT" -o text 2>"$ERR")
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

Verify does not persist any data to disk. Same policy as quorum — no data stored.

---

### continue

**Purpose:** Resume the last Gemini session with a follow-up.

Codex `continue` is NOT supported in v1 (`codex exec resume --last` lacks `--output-last-message`).

**Flow:**
1. If `--via codex` → error: "Continue not supported for codex in v1. Use --via gemini."
2. Extract the follow-up prompt.
3. Execute:
```bash
MODEL=$(_resolve_model "ask" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
ERR=$(mktemp)
RESP=$(gemini $MODEL_FLAG -r latest -p "<follow-up>" -o text 2>"$ERR")
# Standard Gemini error handling (see Error Handling §7) — return 1 on failure
```
Note: Gemini sessions are CWD-scoped.

4. Present the response and your own commentary.

---

### doctor

**Purpose:** Health check — verify all providers are available and authenticated.

**Flow:**
1. **Binary checks** (parallel):
```bash
CLAUDE_BIN=$(which claude 2>/dev/null) && CLAUDE_VER=$(claude --version 2>&1 | head -1)
CODEX_BIN=$(which codex 2>/dev/null) && CODEX_VER=$(codex --version 2>&1 | head -1)
GEMINI_BIN=$(which gemini 2>/dev/null) && GEMINI_VER=$(gemini --version 2>&1 | head -1)
```

2. **Smoke API tests** (parallel, background — three Bash calls with `run_in_background: true`). Note: do NOT use `timeout` command (unavailable on macOS). Use Bash tool's built-in timeout parameter instead:

Claude:
```bash
CLAUDE_MODEL=$(_resolve_model "doctor" "claude")
[ -n "$CLAUDE_MODEL" ] || CLAUDE_MODEL=$(_resolve_model "default" "claude")
[ -n "$CLAUDE_MODEL" ] || CLAUDE_MODEL="sonnet"
OUT=$(mktemp); ERR=$(mktemp)
START=$(date +%s)
CLAUDE_AUTH_JSON=$(env -u ANTHROPIC_API_KEY -u CLAUDECODE claude auth status 2>/dev/null || true)
env -u ANTHROPIC_API_KEY -u CLAUDECODE \
  claude -p --output-format text --model "$CLAUDE_MODEL" \
  "Reply with exactly: ARBITER_HEALTH_OK" >"$OUT" 2>"$ERR"
CLAUDE_API_EXIT=$?
CLAUDE_LATENCY=$(( $(date +%s) - START ))
CLAUDE_API_RESP=$(cat "$OUT")
if [ $CLAUDE_API_EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials\|token\|Invalid API key" "$ERR"; then
    CLAUDE_API_STATUS="auth_expired"
  else
    CLAUDE_API_STATUS="error"
  fi
else
  CLAUDE_API_STATUS="ok"
fi
rm -f "$OUT" "$ERR"
```

Codex:
```bash
CODEX_MODEL=$(_resolve_model "doctor" "codex")
CODEX_MODEL_FLAG=""; [ -n "$CODEX_MODEL" ] && CODEX_MODEL_FLAG="--model $CODEX_MODEL"
OUT=$(mktemp); ERR=$(mktemp)
START=$(date +%s)
codex exec --ephemeral $CODEX_PROFILE_FLAG $CODEX_MODEL_FLAG \
  --output-last-message "$OUT" "Reply with exactly: ARBITER_HEALTH_OK" 2>"$ERR"
CODEX_API_EXIT=$?
CODEX_LATENCY=$(( $(date +%s) - START ))
CODEX_API_RESP=$(cat "$OUT"); rm -f "$OUT" "$ERR"
```

Gemini:
```bash
GEMINI_MODEL=$(_resolve_model "doctor" "gemini")
GEMINI_MODEL_FLAG=""; [ -n "$GEMINI_MODEL" ] && GEMINI_MODEL_FLAG="--model $GEMINI_MODEL"
ERR=$(mktemp)
START=$(date +%s)
GEMINI_API_RESP=$(gemini $GEMINI_MODEL_FLAG -p "Reply with exactly: ARBITER_HEALTH_OK" -o text 2>"$ERR")
GEMINI_API_EXIT=$?
GEMINI_LATENCY=$(( $(date +%s) - START ))
if [ $GEMINI_API_EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials" "$ERR"; then
    GEMINI_API_STATUS="auth_expired"
  else
    GEMINI_API_STATUS="error"
  fi
else
  GEMINI_API_STATUS="ok"
fi
rm -f "$ERR"
```

3. **Determine per-provider status:**
   - Binary missing → `missing` (suggest install URL)
   - Binary found, API timeout → `timeout`
   - Binary found, API auth error → `auth_expired` (suggest `codex auth` / `gemini login`)
   - Binary found, API other error → `error`
   - Binary found, API ok → `healthy`

4. **Config check:**
```bash
_PROVIDERS_CONFIG="${EMPORIUM_PROVIDERS_CONFIG:-$HOME/.claude/emporium-providers.local.md}"
if [ -r "$_PROVIDERS_CONFIG" ]; then
  CONFIG_STATUS="found"
else
  CONFIG_STATUS="not found"
fi
```

5. **Output:**
```
Arbiter Doctor — Health Check

| Provider | Binary     | Version | Model  | API    | Latency |
|----------|------------|---------|--------|--------|---------|
| Claude   | ✓ found    | 2.1.71  | sonnet | ✓ ok   | 3.1s    |
| Codex    | ✓ found    | 0.104   | gpt-5.3-codex | ✓ ok   | 2.3s    |
| Gemini   | ✓ found    | 0.30    | auto-gemini-3 | ✗ auth | —       |

Config: ~/.claude/emporium-providers.local.md (found|not found)

Issues:
- Gemini: auth expired. Run: gemini login
- Claude: nested CLI auth is sanitized automatically by unsetting `ANTHROPIC_API_KEY` and `CLAUDECODE`

Overall: PARTIAL (2/3 providers healthy)
```

If all healthy: "All providers healthy. Ready to orchestrate."
If issues: list each with actionable fix (install URL or auth command).
If both missing: "No external providers found. Install codex or gemini to use multi-provider modes."

---

### diverge

**Purpose:** Generate 3 independent implementations in isolated worktrees using different models and strategy hints, then compare and select the best solution. Use when a non-trivial spec has multiple valid approaches and you want to explore the solution space in parallel before committing.

**All three agents run as background tasks for true parallelism.**

**Flow summary:**
1. Preflight (clean tree check)
2. Freeze spec + context into a Problem Pack
3. Create 3 isolated worktrees (detached, run-scoped)
4. Write prompt files per worktree (with strategy hints)
5. Dispatch Claude (configurable model, default `sonnet`), Codex, and Gemini in parallel
6. Staged evaluation: Pass 1 (stats + cards), Pass 2 (full diff on demand)
7. Decision Matrix via separate anonymized evaluator
8. User selects → merge only the chosen solution

#### 0. Budget Guard

Before starting, check whether diverge is warranted:

```
Trivial heuristic: spec < 50 words AND no architecture keywords AND context_files < 2
```

If ALL three conditions are true → warn:
```
> This spec looks straightforward. Diverge costs 4-6× a normal implement.
> Proceed? (yes / switch to implement)
```

If ANY condition is false, skip the warning and proceed.

#### 1. Preflight

Verify the working tree is clean:

```bash
if [ -n "$(git status --porcelain)" ]; then
  echo "ERROR: Working tree has uncommitted changes."
  echo "Diverge requires a clean tree so all worktrees start from identical state."
  echo "Options: stash / commit / abort"
fi
```

- "stash" → `git stash push -m "diverge: pre-run stash"`, proceed
- "commit" → ask user to commit first, then re-run
- "abort" → stop

Then prune stale worktree registrations from prior crashes:

```bash
git worktree prune
```

#### 2. Problem Pack (Input Freezing)

Freeze all inputs into a single immutable artifact stored outside the repo:

```bash
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"
BASE_COMMIT="$(git rev-parse HEAD)"
RUN_DIR="$(mktemp -d "${TMPDIR:-/tmp}/arbiter-diverge.XXXXXX")"
chmod 700 "$RUN_DIR"  # secure: owner-only access
```

Write `$RUN_DIR/diverge-pack.json`:

```json
{
  "run_id": "20260301T100000Z-12345",
  "spec": "user's implementation spec (verbatim)",
  "context_files": ["list of files read for context"],
  "context_contents": {"path/to/file.py": "frozen file content..."},
  "constraints": "extracted from exploration.md if in signum flow",
  "base_commit": "abc123def",
  "strategies": ["minimal", "refactor", "redesign"],
  "providers": ["claude", "codex", "gemini"],
  "timeout_seconds": 300,
  "lite": false,
  "created_at": "2026-03-01T10:00:00Z"
}
```

Copy the pack into each worktree after creation (untracked files are not inherited by worktrees):

```bash
for LABEL in A B C; do
  mkdir -p "${WT[$LABEL]}/.dev"
  cp "$RUN_DIR/diverge-pack.json" "${WT[$LABEL]}/.dev/diverge-pack.json"
done
```

#### 3. Strategy Hints

**Default strategy set** (rotate across providers to avoid provider-strategy bias):

| Strategy | Hint injected into prompt | Default Provider |
|----------|--------------------------|-----------------|
| `minimal` | "Solve with the smallest possible change. Minimize lines changed, new files, and new dependencies. Patch, don't rebuild." | Claude (sonnet) |
| `refactor` | "Solve by improving the existing code structure. Extract helpers, rename for clarity, improve testability. Moderate change size OK." | Codex (GPT) |
| `redesign` | "Solve with the best possible abstraction. You may introduce new modules, patterns, or interfaces if justified. Optimize for long-term maintainability." | Gemini |

**Custom strategies** (`--strategies "a,b,c"`): injected verbatim as hint text. If fewer than 3 → pad with defaults. If more than 3 → use first 3.

**Provider redistribution:** if a provider is unavailable, redistribute its strategy to an available provider as a second run in a separate worktree. Example: no Gemini → Claude gets `minimal` + `redesign` in two worktrees.

#### 4. Worktree Creation

Worktrees are detached (no branch created upfront — branch is only created for the selected solution at merge time):

```bash
declare -A WT
for LABEL in A B C; do
  WT[$LABEL]="$RUN_DIR/$LABEL"
  git worktree add --detach "${WT[$LABEL]}" "$BASE_COMMIT"
done
```

**Cleanup (trap on exit):**

```bash
cleanup_diverge() {
  for LABEL in A B C; do
    if [ -d "${WT[$LABEL]}" ]; then
      git worktree remove --force "${WT[$LABEL]}" 2>/dev/null
    fi
  done
  rm -rf "$RUN_DIR"
}
trap cleanup_diverge EXIT
```

On any error, remind user: `git worktree list | grep arbiter-diverge` for manual cleanup.

#### 5. Prompt File Generation

Write a prompt file to each worktree (avoids OS argument length limits and shell quoting issues):

```bash
for LABEL in A B C; do
  STRATEGY="${STRATEGIES[$LABEL]}"
  cat > "${WT[$LABEL]}/.dev/diverge-prompt.md" <<PROMPT
# Implementation Task

## Strategy
$STRATEGY

## Specification
$SPEC

## Context Files
$(for f in "${CONTEXT_FILES[@]}"; do
  echo "### $f"
  echo '```'
  cat "$f"
  echo '```'
done)

## Constraints
$CONSTRAINTS

## Rules
- Do NOT commit your changes. Leave all modifications as uncommitted files.
- Write a self-assessment card to .dev/diverge-card.json with this structure:
  {"approach": "5-10 line summary", "assumptions": [...], "risks": [...], "new_dependencies": [...]}
- Follow existing project patterns and conventions.
PROMPT
done
```

#### 6. Agent Dispatch

All three run as background tasks. Agents DO NOT commit — all changes remain as uncommitted modifications in the worktree.

**`run_with_timeout` function** (python process group kill for reliable child cleanup):

```bash
run_with_timeout() {
  local timeout_s="$1"; shift
  python3 -c "
import os, signal, subprocess, sys
timeout = int(sys.argv[1])
cmd = sys.argv[2:]
p = subprocess.Popen(cmd, start_new_session=True)
try:
    rc = p.wait(timeout=timeout)
    sys.exit(rc)
except subprocess.TimeoutExpired:
    try:
        os.killpg(p.pid, signal.SIGTERM)
    except ProcessLookupError:
        pass
    try:
        p.wait(timeout=5)
    except subprocess.TimeoutExpired:
        try:
            os.killpg(p.pid, signal.SIGKILL)
        except ProcessLookupError:
            pass
    sys.exit(124)
" "$timeout_s" "$@"
}
```

Exit code 124 = timeout (matches GNU `timeout` convention). Default `$TIMEOUT` = 300 (configurable via `--timeout`).

**Claude subagent (configurable model, default `sonnet`) — run_in_background: true:**

Resolve Claude model before dispatch:

```bash
CLAUDE_IMPLEMENT_MODEL=$(_resolve_model "implement" "claude")
[ -n "$CLAUDE_IMPLEMENT_MODEL" ] || CLAUDE_IMPLEMENT_MODEL=$(_resolve_model "default" "claude")
[ -n "$CLAUDE_IMPLEMENT_MODEL" ] || CLAUDE_IMPLEMENT_MODEL="sonnet"
```

```
subagent_type="general-purpose", model="$CLAUDE_IMPLEMENT_MODEL", run_in_background=true
Prompt: contents of ${WT[A]}/.dev/diverge-prompt.md
Working directory: ${WT[A]}
```

**Bash call (codex) — run_in_background: true:**

```bash
MODEL=$(_resolve_model "implement" "codex")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
PROMPT_FILE="${WT[B]}/.dev/diverge-prompt.md"
ERR="$RUN_DIR/codex-stderr.log"

# Guard against ARG_MAX — if prompt exceeds 128KB, reference the file instead
PROMPT_SIZE=$(wc -c < "$PROMPT_FILE")
if [ "$PROMPT_SIZE" -gt 131072 ]; then
  CODEX_PROMPT="Read and follow the implementation spec in .dev/diverge-prompt.md"
else
  CODEX_PROMPT=$(cat "$PROMPT_FILE")
fi

run_with_timeout "$TIMEOUT" \
  codex exec --full-auto -C "${WT[B]}" $MODEL_FLAG \
    "$CODEX_PROMPT" \
  >"$RUN_DIR/codex-stdout.log" 2>"$ERR"
CODEX_EXIT=$?

if [ $CODEX_EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials\|token" "$ERR"; then
    echo "Codex auth error. Run: codex auth"
    CODEX_STATUS="auth_expired"
  elif [ $CODEX_EXIT -eq 124 ]; then
    CODEX_STATUS="timeout"
  else
    echo "Codex error (exit $CODEX_EXIT):"; head -5 "$ERR"
    CODEX_STATUS="error"
  fi
else
  CODEX_STATUS="ok"
fi
```

**Bash call (gemini) — run_in_background: true:**

```bash
MODEL=$(_resolve_model "implement" "gemini")
MODEL_FLAG=""; [ -n "$MODEL" ] && MODEL_FLAG="--model $MODEL"
PROMPT_FILE="${WT[C]}/.dev/diverge-prompt.md"
ERR="$RUN_DIR/gemini-stderr.log"

# Pipe prompt via stdin to avoid ARG_MAX / env size limits
run_with_timeout "$TIMEOUT" \
  sh -c "cd '${WT[C]}' && cat '$PROMPT_FILE' | gemini $MODEL_FLAG -y -o text" \
  >"$RUN_DIR/gemini-stdout.log" 2>"$ERR"
GEMINI_EXIT=$?

if [ $GEMINI_EXIT -ne 0 ]; then
  if grep -qi "auth\|login\|403\|401\|credentials" "$ERR"; then
    echo "Gemini auth error. Run: gemini login"
    GEMINI_STATUS="auth_expired"
  elif [ $GEMINI_EXIT -eq 124 ]; then
    GEMINI_STATUS="timeout"
  else
    echo "Gemini error (exit $GEMINI_EXIT):"; head -5 "$ERR"
    GEMINI_STATUS="error"
  fi
else
  GEMINI_STATUS="ok"
fi
```

**NOTE:** Always load the Gemini prompt into a variable first (`PROMPT=$(cat "$PROMPT_FILE")`), then pass via `"$PROMPT"`. Single-quote expansion (`'$(cat ...)'`) blocks variable substitution and will NOT work.

Collect all results using `TaskOutput` tool (block: true) for each background task ID.

Show Run Summary after collecting all dispatch results (see [Run Summary](#run-summary)). On PARTIAL_SUCCESS → redistribute failed provider's strategy or evaluate with fewer solutions. On FAILURE → Error #13 applies.

#### 7. Staged Evaluation

**Problem:** 3 full diffs simultaneously can overflow the evaluator's context (e.g., 3 × 50KB = 150k+ tokens). **Solution:** Two-pass evaluation.

**Pass 1: Stats + Cards**

```bash
for LABEL in A B C; do
  git -C "${WT[$LABEL]}" diff --name-status "$BASE_COMMIT" \
    > "$RUN_DIR/$LABEL-name-status.txt"
  git -C "${WT[$LABEL]}" diff --numstat "$BASE_COMMIT" \
    > "$RUN_DIR/$LABEL-numstat.txt"

  if [ -f "${WT[$LABEL]}/.dev/diverge-card.json" ]; then
    cp "${WT[$LABEL]}/.dev/diverge-card.json" "$RUN_DIR/$LABEL-card.json"
  else
    echo '{"approach":"(agent did not write card)","assumptions":[],"risks":[],"new_dependencies":[]}' \
      > "$RUN_DIR/$LABEL-card.json"
  fi
done
```

Build a Per-Solution Card from stats (labels anonymized as Solution 1/2/3 — provider-solution mapping stored in `$RUN_DIR/label-map.json` but NOT passed to the evaluator):

```json
{
  "label": "1",
  "strategy": "minimal",
  "status": "ok|timeout|error",
  "files_changed": 3,
  "lines_added": 42,
  "lines_removed": 12,
  "files_list": ["src/api.py", "tests/test_api.py"],
  "card": {
    "approach": "agent's self-summary",
    "assumptions": ["..."],
    "risks": ["..."],
    "new_dependencies": []
  }
}
```

**Pass 2: Full Diff on Demand**

Only after the Decision Matrix is presented:
```bash
git -C "${WT[$SELECTED]}" diff "$BASE_COMMIT"
```

#### 8. Test Integration (`--run-tests`)

Before running tests, verify a test suite exists:

```bash
# Detect test runner and set TEST_COMMAND
HAS_TESTS=false
TEST_COMMAND=""
if [ -f "package.json" ] && command -v npm &>/dev/null; then
  HAS_TESTS=true; TEST_COMMAND="npm test"
elif [ -f "pytest.ini" ] || [ -f "setup.cfg" ] || [ -f "pyproject.toml" ] && command -v pytest &>/dev/null; then
  HAS_TESTS=true; TEST_COMMAND="pytest"
elif [ -f "go.mod" ] && command -v go &>/dev/null; then
  HAS_TESTS=true; TEST_COMMAND="go test ./..."
elif ls tests/ &>/dev/null || ls test/ &>/dev/null; then
  # Test files exist but no runner detected
  if command -v pytest &>/dev/null; then HAS_TESTS=true; TEST_COMMAND="pytest"
  elif command -v jest &>/dev/null; then HAS_TESTS=true; TEST_COMMAND="jest"
  fi
fi
```

If `HAS_TESTS=false` or `TEST_COMMAND` is empty → mark `Tests: not run` in decision matrix (do not error).

If test suite found:

```bash
for LABEL in A B C; do
  (cd "${WT[$LABEL]}" && $TEST_COMMAND 2>&1) > "$RUN_DIR/$LABEL-test-results.txt"
  echo "exit:$?" >> "$RUN_DIR/$LABEL-test-results.txt"
done
```

A solution that fails existing tests gets Correctness score capped at 1. Test results are passed to the evaluator as additional input.

**Guard:** `--lite --run-tests` → error: "Cannot run tests in lite mode (no code)." (also caught at input parsing).

#### 9. Decision Matrix (Evaluator)

The evaluator is a **separate Claude invocation** — NOT the orchestrator (mitigates self-preference bias):

Resolve evaluator model before dispatch:

```bash
CLAUDE_EVAL_MODEL=$(_resolve_model "evaluate" "claude")
[ -n "$CLAUDE_EVAL_MODEL" ] || CLAUDE_EVAL_MODEL=$(_resolve_model "default" "claude")
[ -n "$CLAUDE_EVAL_MODEL" ] || CLAUDE_EVAL_MODEL="sonnet"
```

```
subagent_type="general-purpose", model="$CLAUDE_EVAL_MODEL"
run_in_background=false
```

Evaluator prompt receives:
- 3 solution cards (anonymized as Solution 1/2/3)
- 3 file-level diffs (`--name-status`)
- 3 numstats
- The original spec
- **NO provider names, NO model names**

The evaluator produces raw scores per criterion. **Weighted totals are computed programmatically** (not by LLM) via bash arithmetic:

```bash
# Weights: correctness=5, change_size=3, maintainability=4, test_coverage=4,
#          backward_compat=3, performance=2, security=5
WEIGHTS=(5 3 4 4 3 2 5)

for SOL in 0 1 2; do
  TOTAL=0
  for i in "${!WEIGHTS[@]}"; do
    SCORE=$(jq -r ".solutions[$SOL].scores[$i]" "$RUN_DIR/evaluator-output.json")
    TOTAL=$((TOTAL + WEIGHTS[i] * SCORE))
  done
  echo "Solution $((SOL + 1)): $TOTAL"
done
```

**Evaluator overflow fallback:** if evaluator context overflows → fall back to stats/cards only (no diff content).

#### 10. Decision Matrix Output Format

```markdown
## Diverge Report: <spec summary>

**Evaluator:** independent (anonymized labels, formula-computed totals)
**Run ID:** 20260301T100000Z-12345

### Summary

| | Solution 1 | Solution 2 | Solution 3 |
|---|---|---|---|
| Strategy | minimal | refactor | redesign |
| Files changed | 3 | 5 | 8 |
| Lines +/- | +42/-12 | +87/-34 | +156/-67 |
| New deps | 0 | 0 | 1 (p-retry) |
| Tests | PASS | PASS | 2 FAIL |

### Decision Matrix

| Criterion (weight) | Sol 1 | Sol 2 | Sol 3 | Notes |
|---|---|---|---|---|
| Correctness (5) | 4 | 4 | 5 | Sol 3 handles edge case X |
| Change size (3) | 5 | 3 | 1 | Sol 1 is surgical |
| Maintainability (4) | 3 | 5 | 4 | Sol 2 cleanest structure |
| Test coverage (4) | 3 | 4 | 5 | Sol 3 adds most tests |
| Backward compat (3) | 5 | 4 | 3 | Sol 1 preserves all interfaces |
| Performance (2) | 4 | 4 | 4 | No significant difference |
| Security (5) | 4 | 4 | 4 | No issues found |
| **Weighted total** | **99** | **106** | **104** | formula-computed |

### Recommendation

**Solution 2 (refactor)** scores highest. Solution 1 is the safe fallback.

### Options
1. Merge Solution 1 → current branch
2. Merge Solution 2 → current branch
3. Merge Solution 3 → current branch
4. Inspect full diff (specify solution number)
5. Hybrid: cherry-pick files manually (`git restore --source <branch> -- <path>`)
6. Discard all
```

After user selects, reveal the provider mapping:
```
> Solution 1 = Claude (sonnet) | Solution 2 = Codex (GPT) | Solution 3 = Gemini
```

#### 11. Merge

```bash
SELECTED_LABEL="B"  # mapped from user's solution choice
SELECTED_BRANCH="diverge/$RUN_ID-selected"

# HEAD drift check
CURRENT_HEAD="$(git rev-parse HEAD)"
if [ "$CURRENT_HEAD" != "$BASE_COMMIT" ]; then
  echo "WARNING: HEAD moved during diverge ($BASE_COMMIT → $CURRENT_HEAD)."
  echo "Merge may have conflicts. Proceed? (yes / abort)"
fi

# Show files in selected worktree
git -C "${WT[$SELECTED_LABEL]}" status --short

# Create branch only for selected solution
git -C "${WT[$SELECTED_LABEL]}" switch -c "$SELECTED_BRANCH"

# Stage changes, excluding .dev/ artifacts
git -C "${WT[$SELECTED_LABEL]}" add -A -- . ':!.dev/'
git -C "${WT[$SELECTED_LABEL]}" commit -m "diverge: <spec summary> (strategy: <strategy>)"

# Merge into working branch
git merge --no-ff "$SELECTED_BRANCH"

# Cleanup handled by trap
```

Save report to `.dev/diverge-report.md` for audit trail. Append selection metadata.

```bash
mkdir -p .dev
```

Append JSONL metrics artifact (append-only, per-run stats):
```bash
cat >> .dev/diverge-metrics.jsonl <<EOF
{"run_id":"$RUN_ID","selected":"Solution 2","provider":"codex","strategy":"refactor","scores":{...},"selected_at":"$(date -u +%Y-%m-%dT%H:%M:%SZ)"}
EOF
```

#### 12. Lite Mode (`--lite`)

Design-level divergence without code generation. ~1.5× token cost, ~10× faster than full diverge.

Each model produces a **design doc** (architecture + files + risks) instead of code. No worktrees are created — outputs are text artifacts stored in `$RUN_DIR`.

Prompt modification:
```
Do NOT write code. Instead, produce a design document with:
- ## Approach (5-10 lines describing the strategy)
- ## Files (list of files to create/modify with brief notes)
- ## Risks (top 5 risks and mitigations)
- ## Dependencies (new packages needed, if any)
```

Output: side-by-side comparison of 3 designs → user picks one.

**Lite → Implementation handoff** (after user selects a design):

1. Save selected design to `.dev/diverge-selected-design.md`
2. Save full comparison report to `.dev/diverge-report.md`
3. Ask the user:
   ```
   > Selected design saved. How to proceed?
   > 1. signum build — run signum Phase 3 using this design
   > 2. arbiter implement — delegate to a single provider
   > 3. manual — implement yourself
   ```
4. If **signum flow** (`build_strategy=diverge-lite`): overwrite `.dev/design.md` with the selected design, proceed to Phase 3 (standard build, not diverge again)
5. If **standalone**: user decides next step

---

## Run Summary

All multi-provider modes (panel, quorum, verify, diverge) MUST show a Run Summary at the top of their output.

### Format

```markdown
## Run Summary
| Provider | Status | Time |
|----------|--------|------|
| Claude   | ok     | 3.2s |
| Codex    | ok     | 8.1s |
| Gemini   | timeout| 120s |

Overall: PARTIAL_SUCCESS (2/3 providers)
⚠ Gemini timed out after 120s. Results below are from Claude + Codex only.
```

### Provider Status Values

`ok`, `timeout`, `error`, `auth_expired`, `abstain`

### Overall Status (deterministic — computed, not LLM-decided)

- **SUCCESS** — all providers returned `ok`
- **PARTIAL_SUCCESS** — ≥1 `ok` AND ≥1 non-ok
- **FAILURE** — zero `ok` providers

### Behavior

- `SUCCESS` → proceed normally
- `PARTIAL_SUCCESS` → proceed with available results + show warning listing failed providers and reasons
- `FAILURE` → STOP. Show errors. Suggest fallback:
  - panel/verify → "Try `/arbiter ask` with a single provider"
  - quorum → "Cannot form quorum from 0 external votes"
  - diverge → "Try `/arbiter implement` instead"

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
  diverge     3 independent solutions, compare & select
  doctor      Health check all providers

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
  /arbiter diverge "implement caching layer"
  /arbiter diverge --lite "redesign auth module"
  /arbiter diverge --run-tests "add input validation"
  /arbiter doctor

Config: ~/.claude/emporium-providers.local.md (configure models per task)
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
11. **Diverge: dirty tree** → STOP. Ask: stash / commit / abort.
12. **Diverge: provider unavailable** → Skip, redistribute strategy to available provider.
13. **Diverge: all providers fail** → STOP, show errors, suggest `arbiter implement` instead.
14. **Diverge: HEAD drift** → WARN, ask user to confirm merge or abort.
15. **Diverge: evaluator overflow** → Fallback to stats/cards only (no diff content).

## Backward Compatibility

If triggered via `/codex` (old trigger), handle normally — arbiter processes the command. No redirect message needed, just run the requested mode with codex as the provider.

## Rules

1. **Never auto-apply findings.** Always present to user first.
2. **Filter style disagreements** across all providers — different models have different aesthetics.
3. **Show effective command** before execution for transparency.
4. **Cost awareness** — mention if task seems trivial for the selected provider/model.
5. **Worktree cleanup** on error/discard (implement mode). Never leave orphans.
6. **No anchored review** — never show provider A's full answer to provider B for confirmation. Cross-verification must use anonymous claim decomposition (verify mode) or independent generation (panel/quorum). Exception: diverge evaluation compares all solutions equally using anonymized labels (Solution 1/2/3).
7. **Panel: partial failure OK** — if one provider fails, report failure + show others.
8. **Codex review = text** — always parse as plain text, never expect JSON.
9. **Gemini stderr = tempfile** — never `2>/dev/null` blindly, always check for auth errors first.
10. **Quorum: Claude does NOT vote** — only Codex and Gemini vote. Claude moderates and breaks ties.
11. **Quorum: deterministic policy first** — apply decision rules before LLM synthesis. Never override a clear BLOCK without evidence.
12. **Quorum: minimum 1 external** — if both providers fail, abort with error. Never produce a quorum from 0 external votes.
13. **Diverge: no commits.** Agents do not commit. All changes stay uncommitted in worktrees. Diff against base_commit.
14. **Run Summary: always show for multi-provider modes.** Compute overall status deterministically (SUCCESS/PARTIAL_SUCCESS/FAILURE). Never skip the summary.
