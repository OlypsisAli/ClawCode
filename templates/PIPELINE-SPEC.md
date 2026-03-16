# Implementation Pipeline — Operational Runbook

**Purpose:** Standing orders for the pipeline orchestrator. Read this at the start of every pipeline run. If context resets, this file tells you how to resume without improvising.

---

## Overview

You are the **orchestrator**. You do NOT write code directly. You run Codex CLI against the repo, track state, extract issue ledgers, verify artifacts, and escalate when needed.

```
Working Doc -> Setup -> Spec Gen -> Spec Audit Loop -> Implementation -> Build Verify -> Post-Impl Audit Loop -> Finalize
```

---

## Codex CLI Requirements

| Property | Required Value | How to Verify |
|----------|----------------|---------------|
| **Minimum version** | `0.114.0+` | `codex --version` |
| **Binary path** | `$CODEX_BIN` | `which codex` and `ls -la "$CODEX_BIN"` |
| **Config location** | `$CODEX_CONFIG_PATH` | `cat "$CODEX_CONFIG_PATH"` |
| **Auth** | Valid Codex CLI login | `codex exec -m "$MODEL" --json "Say hello"` |

### Required `config.toml` Settings

```toml
model = "gpt-5.3-codex"
model_reasoning_effort = "xhigh"
model_context_window = 1050000  # Example only. Verify the full supported window for the model you actually use.

[model_providers.openai]
name = "OpenAI"
stream_max_retries = 10
stream_idle_timeout_ms = 1200000
request_max_retries = 6

[features]
prevent_idle_sleep = true

[projects."/ABS/PATH/TO/REPO"]
trust_level = "trusted"

[projects."/ABS/PATH/TO/REPO/.worktrees/pipeline"]
trust_level = "trusted"
```

Use `gpt-5.3-codex` as the default long-run baseline. Treat `gpt-5.4` as an optional alternate only if it proves reliable in your auth flow and environment.

### Update Procedure

```bash
npm i -g @openai/codex@latest
ln -sf "$(which codex)" "$CODEX_BIN"
codex --version
```

---

## Paths & Config

```
PIPELINE_DIR=$PIPELINE_DIR
REPO_PATH=$REPO_PATH
WORKTREE_PATH=$WORKTREE_PATH

CODEX_BIN=$CODEX_BIN
CODEX_MIN_VERSION=$CODEX_MIN_VERSION
CODEX_CONFIG_PATH=$CODEX_CONFIG_PATH

MODEL=$MODEL
MODEL_REASONING_EFFORT=$MODEL_REASONING_EFFORT
MODEL_FALLBACK=$MODEL_FALLBACK

PROMPTS_DIR=$PROMPTS_DIR
SCHEMAS_DIR=$SCHEMAS_DIR
STATE_DIR=$STATE_DIR
LOGS_DIR=$LOGS_DIR
INBOX_DIR=$INBOX_DIR

BASE_BRANCH=$BASE_BRANCH
BRANCH_PREFIX=$BRANCH_PREFIX
NOTIFICATION_CHANNEL=$NOTIFICATION_CHANNEL

INSTALL_COMMAND=$INSTALL_COMMAND
BUILD_COMMAND=$BUILD_COMMAND
LINT_COMMAND=$LINT_COMMAND
TEST_COMMAND=$TEST_COMMAND

MAX_SPEC_ITERATIONS=$MAX_SPEC_ITERATIONS
MAX_IMPL_ITERATIONS=$MAX_IMPL_ITERATIONS
```

---

## Pre-Flight Checklist

Run this before every pipeline run. Do not start a run with known toolchain issues.

```bash
# 1. Codex CLI version and path
codex --version
which codex
ls -la "$CODEX_BIN"

# 2. Codex config
cat "$CODEX_CONFIG_PATH"

# 3. Model smoke test
"$CODEX_BIN" exec -m "$MODEL" --json "Say hello"

# 4. Repo health
cd "$REPO_PATH"
git status
git branch --show-current

# 5. Worktree health
git worktree list
# Remove stale pipeline worktree if no run is active:
# git worktree remove "$WORKTREE_PATH" --force
```

If the CLI is outdated or the binary path is stale, update and re-run pre-flight before proceeding.

---

## Worktree Isolation

The pipeline runs in a dedicated worktree at `$WORKTREE_PATH`. This keeps the main checkout clean and gives Codex a stable feature branch to work in.

Rules:
- All `codex exec -C`, build, lint, and test commands run in `$WORKTREE_PATH`.
- Branch creation and worktree lifecycle are managed by the orchestrator, not by Codex.
- Create the worktree in Phase 1. Remove it in Phase 6 after success or after an intentional abort/cleanup.
- Document any required env-file sync commands for your repo. Keep those commands generic in the public template, then customize locally.

---

## Stall Detection & Recovery

### Detecting a Stall

Treat a Codex session as stalled only when all of these are true:
1. The JSONL log has not grown for 5+ minutes.
2. The parent process CPU is 0%.
3. The child `codex` process CPU is 0%.

Short idle periods are normal during long reasoning turns. Do not kill a session just because CPU drops briefly.

### Recovery Procedure

1. Kill the stalled process.
2. Inspect the last JSONL lines to see whether the agent was still reading or had already started writing.
3. Save any useful findings into `context-brief.md`.
4. Re-launch with the right mode:
   - Use fresh `codex exec` for spec generation, first-pass audits, or any turn that needs `-m`, `-C`, `-o`, or `--output-schema`.
   - Use `codex exec resume` only when thread continuity is the main requirement and the saved thread is still valid.
5. After repeated stalls on the same phase:
   - Check/update Codex CLI.
   - Re-check `config.toml` stream settings.
   - Try `$MODEL_FALLBACK`.
   - Escalate if the fallback also fails.

### `exec` vs `exec resume`

| Capability | `codex exec` | `codex exec resume` |
|------------|--------------|---------------------|
| Choose model with `-m` | Yes | No |
| Choose directory with `-C` | Yes | No |
| `--json` output | Yes | Yes |
| `--full-auto` | Yes | Yes |
| `-o <file>` | Yes | No |
| `--output-schema` | Yes | No |

Rule:
- Use fresh `exec` for new phases, stalled first-pass phases, and any command that needs full flags.
- Use `resume` for iterative spec audits, impl audits, and targeted fix turns where preserving the thread improves convergence.

---

## State File

Every feature gets `state/<slug>/state.json`.

```json
{
  "feature_slug": "<slug>",
  "working_doc_path": "state/<slug>/working-doc.md",
  "branch": "feature/<slug>",
  "worktree_path": "/absolute/path/to/repo/.worktrees/pipeline",
  "phase": "setup|spec_gen|spec_audit|implementation|build_verify|impl_audit|completed|failed|needs_human",
  "spec_thread_id": null,
  "spec_audit_thread_id": null,
  "impl_thread_id": null,
  "impl_audit_thread_id": null,
  "spec_iteration": 0,
  "impl_iteration": 0,
  "spec_verdict": null,
  "impl_verdict": null,
  "build_passed": null,
  "lint_passed": null,
  "status": "pending|in_progress|completed|failed|needs_human",
  "error": null,
  "resume_hint": null,
  "started_at": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>"
}
```

Update `resume_hint` every time `phase` or `status` changes. It should say what just happened and what the next session must do.

---

## Phase Execution

### Phase 1: Setup

```bash
cd "$REPO_PATH"
git fetch origin
git checkout "$BASE_BRANCH"
git pull origin "$BASE_BRANCH"

BRANCH="${BRANCH_PREFIX}${SLUG}"
git branch "$BRANCH" "$BASE_BRANCH" 2>/dev/null || true

git worktree remove "$WORKTREE_PATH" --force 2>/dev/null || true
git worktree add "$WORKTREE_PATH" "$BRANCH"

if [ -n "$INSTALL_COMMAND" ]; then
  cd "$WORKTREE_PATH"
  eval "$INSTALL_COMMAND"
fi

# Optional: sync untracked env/config files required by your repo.
# Example:
# ln -sf "$REPO_PATH/.env.local" "$WORKTREE_PATH/.env.local"

mkdir -p "$STATE_DIR/$SLUG"
cp "$WORKING_DOC" "$STATE_DIR/$SLUG/working-doc.md"
# Initialize state.json, including worktree_path="$WORKTREE_PATH"
```

Update state: `phase="setup"`, `status="in_progress"`, `worktree_path="$WORKTREE_PATH"`

Notify immediately:
```text
Starting pipeline for '<feature>' - created branch $BRANCH in isolated worktree
```

### Phase 2: Spec Generation

```bash
JSONL_LOG="$STATE_DIR/$SLUG/spec-gen.jsonl"

"$CODEX_BIN" exec \
  -m "$MODEL" \
  -C "$WORKTREE_PATH" \
  --json \
  --full-auto \
  -o "$STATE_DIR/$SLUG/spec.md" \
  "$(cat "$PROMPTS_DIR/prompt1-spec-writer.md")

---
## WORKING DOC
$(cat "$STATE_DIR/$SLUG/working-doc.md")
---

Read the codebase first.
Generate the implementation spec.
Your FINAL message must be the complete spec content - nothing else." \
  | tee "$JSONL_LOG"

SPEC_THREAD_ID=$(head -1 "$JSONL_LOG" | jq -r '.thread_id')
```

### Phase 3: Spec Audit Loop

Iteration 1 starts a fresh audit thread:

```bash
AUDIT_LOG="$STATE_DIR/$SLUG/audit-1.jsonl"

"$CODEX_BIN" exec \
  -m "$MODEL" \
  -C "$WORKTREE_PATH" \
  --json \
  --full-auto \
  --output-schema "$SCHEMAS_DIR/spec-audit-verdict.json" \
  -o "$STATE_DIR/$SLUG/audit-1.json" \
  "$(cat "$PROMPTS_DIR/prompt2-spec-auditor.md")

---
## WORKING DOC
$(cat "$STATE_DIR/$SLUG/working-doc.md")
---
## SPEC TO AUDIT
$(cat "$STATE_DIR/$SLUG/spec.md")
---

Audit this spec against the working doc and the actual codebase.
Assign stable issue IDs." \
  | tee "$AUDIT_LOG"

SPEC_AUDIT_THREAD_ID=$(head -1 "$AUDIT_LOG" | jq -r '.thread_id')
```

After each audit:

```bash
jq -r '.issues[] | "- [\(.id)] [\(.severity)] [\(.status)] \(.section): \(.issue) -> Fix: \(.suggested_fix)"' \
  "$STATE_DIR/$SLUG/audit-$N.json" > "$STATE_DIR/$SLUG/spec-issues-$N.md"

VERDICT=$(jq -r '.verdict' "$STATE_DIR/$SLUG/audit-$N.json")
BLOCKER_COUNT=$(jq -r '.blocker_count' "$STATE_DIR/$SLUG/audit-$N.json")
```

If `NOT_IMPLEMENTABLE_YET`, revise the spec in the writer thread:

```bash
FIXES=$(jq -r '.issues[] | select(.severity == "critical" or .severity == "major") | select(.status != "resolved") | "- [\(.id)] [\(.severity)] \(.section): \(.issue) -> Fix: \(.suggested_fix)"' \
  "$STATE_DIR/$SLUG/audit-$N.json")

"$CODEX_BIN" exec resume "$SPEC_THREAD_ID" \
  --json \
  --full-auto \
  "Address ONLY these issues:

$FIXES

Return the complete revised spec and write it to $STATE_DIR/$SLUG/spec.md." \
  | tee "$STATE_DIR/$SLUG/spec-fix-$N.jsonl"
```

Re-audits resume the same audit thread. Because `exec resume` cannot use `-o` or `--output-schema`, inline the schema and tell Codex to write the file itself.

Max-iteration rule:
- `blocker_count = 0` -> auto-promote to `IMPLEMENTABLE_WITH_NOTES`
- `blocker_count > 0` -> escalate and stop

### Phase 4: Implementation

```bash
IMPL_LOG="$STATE_DIR/$SLUG/impl.jsonl"

"$CODEX_BIN" exec \
  -m "$MODEL" \
  -C "$WORKTREE_PATH" \
  --json \
  --full-auto \
  "$(cat "$PROMPTS_DIR/implementation-primer.md")

---
## SPEC TO IMPLEMENT
$(cat "$STATE_DIR/$SLUG/spec.md")
---

Implement this spec end-to-end.
Run '$BUILD_COMMAND' and '$LINT_COMMAND' after implementation.
Report changed files in the final message." \
  | tee "$IMPL_LOG"

IMPL_THREAD_ID=$(head -1 "$IMPL_LOG" | jq -r '.thread_id')
```

If the agent reports a spec/codebase mismatch, write `mismatch-report.md`, set `status="needs_human"`, and stop.

### Phase 4.5: Build Verification

```bash
cd "$WORKTREE_PATH"
eval "$BUILD_COMMAND" > "$STATE_DIR/$SLUG/build-log.txt" 2>&1
BUILD_EXIT=$?

eval "$LINT_COMMAND" > "$STATE_DIR/$SLUG/lint-log.txt" 2>&1
LINT_EXIT=$?
```

If either fails, resume the implementation thread with the captured logs and retry.

### Phase 5: Post-Implementation Audit Loop

```bash
cd "$WORKTREE_PATH"
DIFF=$(git diff "$BASE_BRANCH"..HEAD)

POSTAUDIT_LOG="$STATE_DIR/$SLUG/post-audit-1.jsonl"

"$CODEX_BIN" exec \
  -m "$MODEL" \
  -C "$WORKTREE_PATH" \
  --json \
  --full-auto \
  --output-schema "$SCHEMAS_DIR/impl-audit-verdict.json" \
  -o "$STATE_DIR/$SLUG/post-audit-1.json" \
  "$(cat "$PROMPTS_DIR/prompt3-post-audit.md")

---
## SPEC
$(cat "$STATE_DIR/$SLUG/spec.md")
---
## IMPLEMENTATION (Git Diff)
$DIFF
---

Audit this implementation against the spec.
Assign stable compliance and bug IDs." \
  | tee "$POSTAUDIT_LOG"

IMPL_AUDIT_THREAD_ID=$(head -1 "$POSTAUDIT_LOG" | jq -r '.thread_id')
```

After each audit:

```bash
jq -r '
  ([.compliance_issues[]? | "- [\(.id)] [\(.severity)] [\(.status)] COMPLIANCE \(.spec_section): \(.issue) -> Fix: \(.fix)"] +
   [.bugs[]? | "- [\(.id)] [\(.severity)] [\(.status)] BUG \(.file // "unknown"): \(.description) -> Fix: \(.fix)"])
  | join("\n")
' "$STATE_DIR/$SLUG/post-audit-$N.json" > "$STATE_DIR/$SLUG/impl-issues-$N.md"
```

If `NOT_COMPLIANT`, resume the implementation thread with only unresolved blocker fixes. Re-audits stay in the same audit thread.

Max-iteration rule:
- `blocker_count = 0` -> auto-promote to `COMPLIANT_WITH_NOTES`
- `blocker_count > 0` -> escalate and stop

### Phase 6: Finalize

```bash
cd "$WORKTREE_PATH"

git add -A -- \
  ':(exclude).env*' \
  ':(exclude)*.log' \
  ':(exclude)node_modules' \
  ':(exclude).DS_Store'

git commit -m "feat($SLUG): implement feature from working doc"
git push origin "$BRANCH"

git diff --name-only "$BASE_BRANCH"..HEAD > "$STATE_DIR/$SLUG/files-changed.txt"

cd "$REPO_PATH"
git worktree remove "$WORKTREE_PATH" --force 2>/dev/null || true
```

Generate `summary.md`, update `state.json` to completed, and send the completion message immediately.

---

## Error Handling

Escalate when:
- Iteration caps are reached with blockers remaining
- The same blocking verdict repeats without progress
- Git operations fail
- Codex auth/model errors prevent forward progress
- Worktree setup or cleanup fails
- The implementation reports a codebase mismatch

Preserve artifacts and include the state file path in the escalation message.

---

## Communication

Use the OpenClaw `message` tool for milestone notifications.

Send proactively at:
- Pipeline start
- Spec generated
- Spec audit passed
- Implementation complete
- Build/lint passed
- Impl audit passed
- Escalation
- Final completion summary

Do not bury milestone notifications inside heartbeat responses.

---

## Concurrency

Default mode is a single shared worktree at `$WORKTREE_PATH`, so run one pipeline at a time unless you deliberately add multiple worktree paths and isolate state for each.

---

## Session Startup Checklist

When a new session or heartbeat takes over:
1. Read this file.
2. Inspect `state/` for in-progress features.
3. Read `state.json` and `resume_hint`.
4. Reconstruct context from `context-brief.md` and the latest artifacts.
5. Resume with the correct mode: fresh `exec` or `exec resume`.
6. Re-send milestone or escalation notifications if they have not already been sent.

---

## Key Rules

1. I am the orchestrator, not the coder.
2. Pre-flight is mandatory before every run.
3. The pipeline runs in the isolated worktree, not the main checkout.
4. Issue ledgers drive every audit/fix loop.
5. `blocker_count` decides whether the pipeline can proceed.
6. `exec resume` is limited; do not use it where fresh `exec` is required.
7. Update `state.json` and `resume_hint` after every meaningful state change.
8. Milestone notifications are proactive, not heartbeat footnotes.
