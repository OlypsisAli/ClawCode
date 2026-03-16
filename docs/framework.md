# Automated Code Implementation Pipeline — Complete Setup Guide

**Purpose:** This document describes an end-to-end automated pipeline that takes a working doc and produces an implemented, audited feature branch with minimal human intervention.

**Audience:** Another OpenClaw agent that needs to stand up and operate this pipeline for its human.

**Scope:** Public framework guidance for OpenClaw users. Replace placeholders with your own repo paths, commands, and environment constraints. Do not copy personal paths from private runbooks.

---

## Table of Contents

1. [How It Works (Big Picture)](#1-how-it-works-big-picture)
2. [Architecture & Roles](#2-architecture--roles)
3. [Codex CLI Requirements & Verification](#3-codex-cli-requirements--verification)
4. [Step-by-Step Setup Checklist](#4-step-by-step-setup-checklist)
5. [Directory Structure & Setup](#5-directory-structure--setup)
6. [Configuration File](#6-configuration-file)
7. [The 6-Phase Pipeline](#7-the-6-phase-pipeline)
8. [State Management](#8-state-management)
9. [Prompt Files](#9-prompt-files)
10. [Schema Files](#10-schema-files)
11. [Verdict System & Iteration Logic](#11-verdict-system--iteration-logic)
12. [JSONL Parsing Reference](#12-jsonl-parsing-reference)
13. [Error Handling & Escalation](#13-error-handling--escalation)
14. [Notification Wiring](#14-notification-wiring)
15. [AGENTS.md & HEARTBEAT.md Integration](#15-agentsmd--heartbeatmd-integration)
16. [Heartbeat Recovery Integration](#16-heartbeat-recovery-integration)
17. [Concurrency](#17-concurrency)
18. [Operational Runbook Template](#18-operational-runbook-template)
19. [Example Working Doc](#19-example-working-doc)
20. [Step-by-Step Execution Checklist](#20-step-by-step-execution-checklist)
21. [Lessons Learned & Gotchas](#21-lessons-learned--gotchas)
22. [Adapting to Your Project](#22-adapting-to-your-project)

---

## 1. How It Works (Big Picture)

```
Human writes a working doc
        ↓
  [Phase 1] Setup — base branch sync, feature branch, isolated worktree, state init
        ↓
  [Phase 2] Spec Generation — Codex reads codebase + working doc -> implementation spec
        ↓
  [Phase 3] Spec Audit Loop — audit spec, extract ledger, revise, re-audit
        ↓
  [Phase 4] Implementation — Codex implements the approved spec inside the worktree
        ↓
  [Phase 4.5] Build Verification — run install/build/lint/test commands as configured
        ↓
  [Phase 5] Post-Impl Audit Loop — audit code, extract ledger, fix, re-audit
        ↓
  [Phase 6] Finalize — summary, push feature branch, clean up worktree
        ↓
Human reviews and merges
```

The current production pattern is deliberate:

- OpenClaw is the orchestrator, not the coder.
- Codex CLI is the coding backend.
- Every phase writes artifacts to disk.
- Every code-writing phase runs in `.worktrees/pipeline`, not the user's main checkout.
- Every audit loop is constrained by issue ledgers and `blocker_count`.

---

## 2. Architecture & Roles

### The Orchestrator (OpenClaw)

- Reads this framework doc and the generated `PIPELINE-SPEC.md`
- Runs pre-flight checks before every pipeline run
- Creates the worktree and feature branch
- Launches Codex CLI sessions
- Parses JSONL output and structured verdict files
- Tracks state in `state/<slug>/state.json`
- Sends milestone notifications proactively
- Escalates when the pipeline stops converging or the environment is broken

### The Coding Agent (Codex CLI)

- Reads the repo from the isolated worktree
- Writes the implementation spec
- Audits specs and code
- Implements code changes
- Runs verification commands
- Writes structured JSON when given a schema

### The Human

- Writes or approves the working doc
- Reviews escalation notices
- Reviews the resulting branch
- Merges the feature

### Separation of Concerns

The orchestrator owns process and Git lifecycle. The coding agent owns code understanding and code changes. Do not blur those roles. If the orchestrator starts improvising code or the coding agent starts managing branches, the pipeline becomes harder to reason about and harder to recover.

---

## 3. Codex CLI Requirements & Verification

### Required Software

- **OpenClaw** with an AI model configured for orchestration
- **Codex CLI** `0.114.0+`
- **Git**
- **jq**
- **Your project toolchain** for install/build/lint/test commands

### Current Model Guidance

ClawCode no longer treats `codex-3.6-extra-high` as the recommended production model. The current practical guidance is:

- Start with `gpt-5.3-codex` as the proven stable baseline for long pipeline runs
- Only prefer `gpt-5.4` if your Codex CLI auth flow exposes it **and** it proves reliable in your environment
- Verify model availability with a smoke test before every run
- Do not assume a model is usable just because a private runbook used it recently

Model availability changes by CLI version, auth flow, and account tier. Treat model selection as a runtime verification step, not tribal knowledge.

### Required `~/.codex/config.toml` Guidance

Use this as the baseline:

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

Why these fields matter:

- `stream_max_retries`: reduces failures on long reasoning turns
- `stream_idle_timeout_ms`: gives xhigh reasoning enough time before the stream is declared dead
- `request_max_retries`: improves resilience around transient transport/API failures
- `model_context_window`: must match the model you actually plan to run

### Verification Commands

Run these during setup and before every pipeline run:

```bash
codex --version
which codex
ls -la ~/.local/bin/codex
cat ~/.codex/config.toml
codex exec -m "gpt-5.3-codex" --json "Say hello"
```

If `~/.local/bin/codex` points at an old binary, fix it:

```bash
npm i -g @openai/codex@latest
ln -sf "$(which codex)" ~/.local/bin/codex
codex --version
```

The stale symlink failure mode is real. Production stalls repeatedly traced back to "Codex was updated, but the pipeline still called an old binary."

---

## 4. Step-by-Step Setup Checklist

Follow this in order.

### Phase A: Install and Verify Dependencies

- [ ] Verify Git: `git --version`
- [ ] Verify jq: `jq --version`
- [ ] Install or update Codex CLI: `npm i -g @openai/codex@latest`
- [ ] Verify the binary path: `which codex` and `ls -la ~/.local/bin/codex`
- [ ] Authenticate Codex CLI with the login flow your environment uses
- [ ] Run a smoke test: `codex exec -m <model> --json "Say hello"`
- [ ] Open `~/.codex/config.toml` and confirm the stream retry/timeout settings are present

### Phase B: Create the Pipeline Directory

- [ ] Choose a pipeline directory, for example `<workspace>/pipeline/`
- [ ] Create `prompts/`, `schemas/`, `state/`, `logs/`, and `inbox/`
- [ ] Create `config.yaml` using Section 6 or [templates/config.yaml](../templates/config.yaml)

### Phase C: Define the Worktree Strategy

- [ ] Use a dedicated worktree path: `<repo>/.worktrees/pipeline`
- [ ] Add trust entries in `~/.codex/config.toml` for both the repo root and the worktree path
- [ ] Decide how your repo will sync untracked env/config files into the worktree, if needed
- [ ] Decide which install/bootstrap command should run immediately after worktree creation

### Phase D: Create Prompt and Schema Files

- [ ] Copy the prompt files from `prompts/`
- [ ] Copy the schemas from `schemas/`
- [ ] Customize `prompts/implementation-primer.md` for your environment

### Phase E: Create the Operational Runbook

- [ ] Copy [templates/PIPELINE-SPEC.md](../templates/PIPELINE-SPEC.md) into your pipeline directory
- [ ] Replace every placeholder with actual values from your config
- [ ] Add repo-specific env sync commands where the template tells you to
- [ ] Review every phase command before the first run

### Phase F: Wire into OpenClaw

- [ ] Add pipeline instructions to `AGENTS.md`
- [ ] Add heartbeat recovery logic to `HEARTBEAT.md`
- [ ] Verify the `message` tool can send notifications on your chosen channel

### Phase G: Dry Run

- [ ] Write a small working doc
- [ ] Run the full pipeline once
- [ ] Update your `implementation-primer.md` and `PIPELINE-SPEC.md` with environment lessons
- [ ] Keep those lessons in the runbook, not only in transient memory

---

## 5. Directory Structure & Setup

You are managing two roots: the repo and the pipeline directory.

```text
<repo>/
├── .worktrees/
│   └── pipeline/              # Created in Phase 1, removed in Phase 6
└── ... project files ...

<pipeline_dir>/
├── PIPELINE-SPEC.md
├── config.yaml
├── prompts/
├── schemas/
├── logs/
├── inbox/
└── state/
    └── <feature-slug>/
        ├── state.json
        ├── working-doc.md
        ├── spec.md
        ├── spec-gen.jsonl
        ├── audit-N.json
        ├── audit-N.jsonl
        ├── spec-issues-N.md
        ├── spec-fix-N.jsonl
        ├── impl.jsonl
        ├── build-log.txt
        ├── lint-log.txt
        ├── post-audit-N.json
        ├── post-audit-N.jsonl
        ├── impl-issues-N.md
        ├── fix-N.jsonl
        ├── mismatch-report.md
        ├── context-brief.md
        └── summary.md
```

The repo worktree is disposable. The pipeline state directory is the durable record.

---

## 6. Configuration File

Use [templates/config.yaml](../templates/config.yaml) as the starting point. A current baseline looks like this:

```yaml
pipeline_dir: /path/to/your/pipeline
repo_path: /path/to/your/repo
worktree_path: /path/to/your/repo/.worktrees/pipeline

codex_bin: ~/.local/bin/codex
codex_min_version: 0.114.0
codex_config_path: ~/.codex/config.toml

model: gpt-5.3-codex
model_reasoning_effort: xhigh
model_fallback: gpt-5.4

max_spec_iterations: 3
max_impl_iterations: 3

state_dir: /path/to/your/pipeline/state
prompts_dir: /path/to/your/pipeline/prompts
schemas_dir: /path/to/your/pipeline/schemas
logs_dir: /path/to/your/pipeline/logs
inbox_dir: /path/to/your/pipeline/inbox

notification_channel: webchat
base_branch: staging
branch_prefix: feature/

install_command: "npm install"
build_command: "npm run build"
lint_command: "npm run lint"
test_command: "npm test"
```

Notes:

- `worktree_path` is now part of the core configuration, not an optional afterthought.
- `model_fallback` matters. It is the first thing you try after repeated stalls on the primary model.
- Keep repo-specific env sync steps out of this public template if they require private paths or private filenames. Document them in your local runbook.

---

## 7. The 6-Phase Pipeline

### Pre-Flight Checklist (Mandatory Before Every Run)

Run this checklist every time:

```bash
# 1. Codex CLI version and path
codex --version
which codex
ls -la "$CODEX_BIN"

# 2. Codex config
cat "$CODEX_CONFIG_PATH"

# 3. Model smoke test
"$CODEX_BIN" exec -m "$MODEL" --json "Say hello"

# 4. Repo state
cd "$REPO_PATH"
git status
git branch --show-current

# 5. Worktree health
git worktree list
```

If any of these fail, fix them before starting a feature run. Production experience says otherwise you waste hours debugging symptoms instead of causes.

### Worktree Isolation Rules

- All `codex exec -C` commands target `$WORKTREE_PATH`
- All build, lint, and test commands run from `$WORKTREE_PATH`
- Git branch creation and worktree lifecycle happen from `$REPO_PATH`
- Create the worktree in Phase 1
- Remove the worktree in Phase 6
- Sync untracked env/config files into the worktree only if your repo requires them

### Stall Detection & Recovery

Treat a session as stalled only when all of these are true:

1. JSONL file has not grown in 5+ minutes
2. Parent process CPU is 0%
3. Child `codex` process CPU is 0%

Recovery:

1. Kill the stalled process
2. Inspect the last JSONL lines
3. Save any useful findings into `context-brief.md`
4. Re-launch with the right mode:
   - Fresh `exec` for spec generation, first-pass audits, or any phase that needs `-m`, `-C`, `-o`, or `--output-schema`
   - `exec resume` for iterative audits and targeted fix turns when the thread is still valid
5. After repeated stalls on the same phase:
   - Update Codex CLI
   - Re-check `config.toml`
   - Try `MODEL_FALLBACK`
   - Escalate if the fallback also fails

### `exec` vs `exec resume`

| Capability | `codex exec` | `codex exec resume` |
|------------|--------------|---------------------|
| Pick model with `-m` | Yes | No |
| Pick directory with `-C` | Yes | No |
| `--json` | Yes | Yes |
| `--full-auto` | Yes | Yes |
| `-o` | Yes | No |
| `--output-schema` | Yes | No |

Use `resume` because you want continuity, not because you want convenience. It is deliberately more limited.

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

# Optional: repo-specific env sync commands go here

mkdir -p "$STATE_DIR/$SLUG"
cp "$WORKING_DOC" "$STATE_DIR/$SLUG/working-doc.md"
```

Initialize `state.json` with:

- `branch`
- `worktree_path`
- `phase: "setup"`
- `status: "in_progress"`
- timestamps
- `resume_hint`

Send the start notification immediately.

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
```

Extract and save `spec_thread_id` from the first JSONL line.

### Phase 3: Spec Audit Loop

Iteration 1:

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
```

After every audit:

```bash
jq -r '.issues[] | "- [\(.id)] [\(.severity)] [\(.status)] \(.section): \(.issue) -> Fix: \(.suggested_fix)"' \
  "$STATE_DIR/$SLUG/audit-$N.json" > "$STATE_DIR/$SLUG/spec-issues-$N.md"

VERDICT=$(jq -r '.verdict' "$STATE_DIR/$SLUG/audit-$N.json")
BLOCKER_COUNT=$(jq -r '.blocker_count' "$STATE_DIR/$SLUG/audit-$N.json")
```

If the verdict is `NOT_IMPLEMENTABLE_YET`, resume the writer thread with only unresolved blocker issues.

Re-audits resume the same audit thread. Because `exec resume` does not support `-o` or `--output-schema`, include the schema inline and instruct Codex to write the JSON verdict file itself.

Max-iteration behavior:

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
```

If the implementation reports a spec/codebase mismatch:

- write `mismatch-report.md`
- set `status: "needs_human"`
- notify the human immediately
- stop until the spec or repo assumptions are corrected

### Phase 4.5: Build Verification

```bash
cd "$WORKTREE_PATH"
eval "$BUILD_COMMAND" > "$STATE_DIR/$SLUG/build-log.txt" 2>&1
BUILD_EXIT=$?

eval "$LINT_COMMAND" > "$STATE_DIR/$SLUG/lint-log.txt" 2>&1
LINT_EXIT=$?
```

If either command fails, resume the implementation thread with the captured logs and retry. This counts toward implementation iterations.

### Phase 5: Post-Implementation Audit Loop

Iteration 1:

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
```

After every audit:

```bash
jq -r '
  ([.compliance_issues[]? | "- [\(.id)] [\(.severity)] [\(.status)] COMPLIANCE \(.spec_section): \(.issue) -> Fix: \(.fix)"] +
   [.bugs[]? | "- [\(.id)] [\(.severity)] [\(.status)] BUG \(.file // "unknown"): \(.description) -> Fix: \(.fix)"])
  | join("\n")
' "$STATE_DIR/$SLUG/post-audit-$N.json" > "$STATE_DIR/$SLUG/impl-issues-$N.md"
```

If the verdict is `NOT_COMPLIANT`, resume the implementation thread with unresolved blocker fixes only. Re-audits resume the same audit thread.

Max-iteration behavior:

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

Generate `summary.md`, mark the pipeline completed in state, and send the completion notification immediately. Do not hide the final summary in a heartbeat reply.

---

## 8. State Management

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

Rules:

- `worktree_path` is mandatory
- `resume_hint` is mandatory after every phase or status change
- `state.json` is the source of truth for recovery
- if it is not on disk, it did not happen

Generate `context-brief.md` between major phases. It should summarize what is done, what was learned, what to avoid, and what remains.

---

## 9. Prompt Files

The extracted prompt files live in [prompts/](../prompts/):

| File | Purpose |
|------|---------|
| `prompt0-working-doc.md` | Optional helper to create a stronger working doc |
| `prompt1-spec-writer.md` | Generates the implementation spec |
| `prompt2-spec-auditor.md` | Audits the spec against the working doc and codebase |
| `prompt3-post-audit.md` | Audits the implementation against the spec |
| `implementation-primer.md` | Prepended to the spec for implementation turns |

Maintenance rules:

- Keep the prompts generic and repo-portable
- Put repo-specific environment constraints in the deployed copy of `implementation-primer.md`
- Do not let the implementation primer tell Codex to manage branch creation or worktree lifecycle; that is the orchestrator's job now

---

## 10. Schema Files

The extracted schema files live in [schemas/](../schemas/):

| File | Purpose |
|------|---------|
| `spec-audit-verdict.json` | Structured output for spec audit |
| `impl-audit-verdict.json` | Structured output for implementation audit |

Both schemas rely on the same core ideas:

- stable issue IDs
- `status` per issue (`new`, `unresolved`, `resolved`)
- severity tiers
- `blocker_count`

The orchestrator should make decisions from `blocker_count`, not from raw issue count.

---

## 11. Verdict System & Iteration Logic

### Severity Tiers

| Severity | Blocks Pipeline? | Meaning |
|----------|------------------|---------|
| `critical` | Yes | Security hole, data loss, or broken core flow |
| `major` | Yes | Significant gap that must be fixed before proceeding |
| `minor` | No | Improvement or edge case |
| `info` | No | Observation or out-of-scope note |

### Verdict Tiers

| Spec Audit | Impl Audit | Meaning |
|------------|------------|---------|
| `IMPLEMENTABLE` | `COMPLIANT` | No blocking issues |
| `IMPLEMENTABLE_WITH_NOTES` | `COMPLIANT_WITH_NOTES` | Only minor/info issues remain |
| `NOT_IMPLEMENTABLE_YET` | `NOT_COMPLIANT` | Blocking issues remain |
| `ESCALATE` | `ESCALATE` | Human judgment or intervention required |

### Issue Ledgers Drive Convergence

This is non-negotiable.

1. After each audit, extract a ledger file
2. Feed the ledger to the fix turn
3. Feed the same ledger to the re-audit turn
4. Only allow new issues if the revision introduced them
5. Preserve stable IDs across iterations

Without ledgers, audits expand scope and do not converge.

### Max Iteration Auto-Promotion

When the iteration cap is reached:

- `blocker_count = 0` -> auto-promote to the appropriate `WITH_NOTES` verdict and proceed
- `blocker_count > 0` -> escalate and stop

Keep this logic. It is one of the core production safeguards.

---

## 12. JSONL Parsing Reference

`codex exec --json` emits newline-delimited JSON.

Common events:

```jsonc
{"type":"thread.started","thread_id":"<uuid>"}
{"type":"turn.started"}
{"type":"turn.completed","usage":{"input_tokens":0,"output_tokens":0}}
{"type":"turn.failed","error":{"message":"..."}}
{"type":"item.completed","item":{"type":"agent_message","text":"..."}}
{"type":"item.completed","item":{"type":"command_execution","command":"...","exit_code":0}}
{"type":"error","message":"..."}
```

Useful extractions:

```bash
THREAD_ID=$(head -1 "$JSONL_LOG" | jq -r '.thread_id')
LAST_MESSAGE=$(grep '"type":"agent_message"' "$JSONL_LOG" | tail -1 | jq -r '.item.text')
TURN_FAILURE=$(grep '"turn.failed"' "$JSONL_LOG" | jq -r '.error.message')
```

### Important `exec resume` Limitations

`codex exec resume` does not support:

- `-C`
- `-m`
- `-o`
- `--output-schema`

That is why re-audits must inline schemas and tell Codex to write files directly.

---

## 13. Error Handling & Escalation

### Recoverable Errors

| Error | Action |
|-------|--------|
| Non-zero Codex exit | Retry once after checking logs |
| JSONL parse error | Retry once if critical fields are missing |
| `turn.failed` | Extract error, retry once |
| Missing `thread_id` | Start a fresh session and record the discontinuity |
| Build/lint failure | Resume implementation thread with the captured logs |
| Stalled session | Kill, inspect, re-launch with the correct mode |

### Escalation Triggers

Escalate when:

- max iterations are reached with blockers remaining
- the same blocking verdict repeats without progress
- Codex auth/model availability prevents forward progress
- worktree setup or cleanup fails
- Git operations fail
- the implementation reports a codebase mismatch
- a fallback model also stalls

### Escalation Message Template

```text
Pipeline needs attention: <feature>

Phase: <phase>
Issue: <description>
Branch: <branch>
State file: state/<slug>/state.json

Options:
1. Fix manually and tell me to resume from phase <next_phase>
2. Tell me to retry the current phase
3. Tell me to abort
```

Send it immediately through the notification channel.

---

## 14. Notification Wiring

Use OpenClaw's `message` tool:

```text
message(
  action="send",
  channel="<notification_channel>",
  message="<text>"
)
```

Send notifications proactively at:

- pipeline start
- spec generated
- spec audit passed
- implementation complete
- build/lint passed
- impl audit passed
- escalation
- pipeline complete

Critical rule: milestone notifications should be sent proactively, not buried in heartbeat responses.

---

## 15. AGENTS.md & HEARTBEAT.md Integration

Add pipeline instructions to `AGENTS.md` so the orchestrator knows what to do when a human asks for a run:

```markdown
### Pipeline Sessions
If the human asks you to run the implementation pipeline:
- Read `pipeline/PIPELINE-SPEC.md`
- Run pre-flight before the feature starts
- Follow Phase 1 through Phase 6 in order
- Update `state.json` and `resume_hint` after every meaningful state change
- Send milestone notifications immediately
- Escalate instead of improvising when the runbook says to stop
```

The exact phrasing can vary, but those operating rules should not.

---

## 16. Heartbeat Recovery Integration

Add recovery logic to `HEARTBEAT.md`:

```markdown
## Priority 1: Pipeline Status Check
Check `pipeline/state/` for in-progress features.
For each feature:
- Read `state.json`
- Read `resume_hint`
- Read `context-brief.md` if it exists
- Determine whether the saved Codex session should be resumed or restarted fresh
- Re-read `pipeline/PIPELINE-SPEC.md` before taking action
- If the run needs human input, re-send the escalation notification if necessary
```

Recovery rules:

- if the session is still healthy, keep waiting or poll it
- if the session stalled, use the stall recovery procedure
- if the session is gone but the phase is resumable, restart with the correct mode
- do not reply with a generic heartbeat acknowledgement if you are actually resuming work

---

## 17. Concurrency

The default framework uses a single worktree path: `.worktrees/pipeline`.

That means:

- one pipeline run at a time by default
- the user's main checkout remains free for normal work
- feature state remains isolated in `state/<slug>/`

You can extend this later with multiple worktrees such as `.worktrees/pipeline-1` and `.worktrees/pipeline-2`, but treat that as a deliberate expansion, not the default.

---

## 18. Operational Runbook Template

The extracted template is [templates/PIPELINE-SPEC.md](../templates/PIPELINE-SPEC.md). That file should be copied into the deployed pipeline directory and placeholder-substituted.

The runbook must cover:

1. Codex CLI requirements and verification
2. `config.toml` guidance
3. paths and config variables
4. pre-flight checklist
5. worktree isolation rules
6. stall detection and recovery
7. `exec` vs `exec resume` guidance
8. state file shape including `worktree_path`
9. Phase 1 through Phase 6 commands
10. escalation rules
11. notification rules
12. session startup and recovery rules

If you update the operational behavior of the framework, update this doc first, then sync the extracted template.

---

## 19. Example Working Doc

```markdown
# Add User Notification Preferences

## Goal
Allow users to control which notifications they receive and on which channels.

## Context
- Users report notification spam
- Notifications are currently sent through one shared function
- No preference model exists yet

## Current State
- `src/lib/notifications.ts` sends notifications directly
- `src/app/api/notifications/route.ts` triggers that flow
- No `notification_preferences` table exists

## How It Should Work
1. Add a `notification_preferences` data model
2. Add GET/PUT API endpoints for preferences
3. Add UI controls on the settings page
4. Check preferences before sending notifications

## Technical Approach
- Reuse the existing settings page layout
- Follow the project's current API and data-access patterns
- Add tests for enabled, disabled, and missing-preference cases

## What NOT to Change
- Do not replace the notification delivery providers
- Do not add notification batching or scheduling

## Scope
- IN: preference storage, API, UI, send-time filtering
- OUT: templates, delivery infrastructure, notification history

## Edge Cases
- Missing preference row defaults to enabled
- Deleted user deletes preferences
- Concurrent updates can be last-write-wins
```

The exact feature does not matter. What matters is that the working doc clearly defines scope, current state, desired behavior, and non-goals.

---

## 20. Step-by-Step Execution Checklist

### Pre-Flight

- [ ] Read `PIPELINE-SPEC.md`
- [ ] Read the working doc
- [ ] Derive the feature slug
- [ ] Run the full pre-flight checklist

### Phase 1: Setup

- [ ] Sync the base branch
- [ ] Create or confirm the feature branch
- [ ] Create the isolated worktree
- [ ] Run the install/bootstrap command
- [ ] Sync required untracked env/config files if needed
- [ ] Copy the working doc into state
- [ ] Initialize `state.json` with `worktree_path`
- [ ] Update `resume_hint`
- [ ] Send the start notification

### Phase 2: Spec Generation

- [ ] Run fresh `codex exec`
- [ ] Save `spec.md`
- [ ] Save the JSONL log
- [ ] Save `spec_thread_id`
- [ ] Update state and `resume_hint`
- [ ] Send the spec-generated notification

### Phase 3: Spec Audit Loop

- [ ] Run fresh audit for iteration 1
- [ ] Resume the same audit thread for re-audits
- [ ] Extract `spec-issues-N.md`
- [ ] Parse `verdict` and `blocker_count`
- [ ] If needed, resume the writer thread with blocker fixes only
- [ ] Auto-promote only when max iterations are hit and `blocker_count = 0`
- [ ] Update state after every iteration
- [ ] Send the audit-passed or escalation notification

### Phase 4: Implementation

- [ ] Run fresh `codex exec` in the worktree
- [ ] Save `impl_thread_id`
- [ ] If the agent reports a mismatch, stop and escalate
- [ ] Update state and `resume_hint`
- [ ] Send the implementation-complete notification

### Phase 4.5: Build Verification

- [ ] Run build and lint commands from the worktree
- [ ] If either fails, resume the implementation thread with the logs
- [ ] Re-run verification after fixes
- [ ] Update state and `resume_hint`
- [ ] Send the build-passed notification

### Phase 5: Post-Implementation Audit Loop

- [ ] Generate the diff against the base branch
- [ ] Run fresh audit for iteration 1
- [ ] Resume the same audit thread for re-audits
- [ ] Extract `impl-issues-N.md`
- [ ] Parse `verdict` and `blocker_count`
- [ ] If needed, resume the implementation thread with blocker fixes only
- [ ] Auto-promote only when max iterations are hit and `blocker_count = 0`
- [ ] Update state after every iteration
- [ ] Send the audit-passed or escalation notification

### Phase 6: Finalize

- [ ] Commit on the feature branch
- [ ] Push the feature branch
- [ ] Generate `summary.md`
- [ ] Remove the worktree
- [ ] Mark the state completed
- [ ] Send the final summary immediately

---

## 21. Lessons Learned & Gotchas

1. **Outdated Codex CLI versions cause stalls.** Check before every run.
2. **`stream_idle_timeout_ms` matters.** Long reasoning turns need more patience than the default.
3. **The stale binary path problem is real.** `which codex` and `ls -la ~/.local/bin/codex` belong in pre-flight.
4. **Worktree isolation is worth it.** It keeps the main checkout clean and avoids accidental branch interference.
5. **`exec resume` is not a full replacement for fresh `exec`.** Its limitations are operationally important.
6. **Issue ledgers are the difference between convergence and thrash.**
7. **`blocker_count` is the right gate.** Total issue count is not.
8. **`WITH_NOTES` auto-promotion should stay.** It prevents minor issues from deadlocking the pipeline at iteration caps.
9. **Notifications need to be explicit and timely.** A finished run hidden inside a heartbeat is effectively invisible.
10. **State is king.** `worktree_path`, verdicts, and `resume_hint` must be on disk.

---

## 22. Adapting to Your Project

Customize these safely:

- `install_command`, `build_command`, `lint_command`, `test_command`
- base branch and branch prefix
- notification channel
- env sync commands for the worktree
- environment constraints inside the implementation primer
- model selection and fallback, after verification

Do not remove these framework invariants unless you have a strong reason:

- pre-flight before every run
- isolated worktree flow
- issue-ledger audit loops
- `blocker_count` gating
- `WITH_NOTES` auto-promotion
- proactive milestone notifications
- explicit state and recovery behavior

Those are the parts that came from production failures, not theory.
