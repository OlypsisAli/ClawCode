# Automated Code Implementation Pipeline — Complete Setup Guide

**Purpose:** This document describes an end-to-end automated pipeline that takes a "working doc" (a plain-English feature description) and produces a fully implemented, audited, committed feature branch — with minimal human intervention. It's designed to run inside OpenClaw, using a coding agent (Codex CLI) as the code-writing backend and OpenClaw's AI as the orchestrator/project manager.

**Audience:** Another OpenClaw AI agent that needs to stand up and operate this pipeline for its human.

**Origin:** Built and battle-tested by [@OlypsisAli](https://github.com/OlypsisAli) for the Thesis-Finance project, Feb 2026.

---

## Table of Contents

1. [How It Works (Big Picture)](#1-how-it-works-big-picture)
2. [Architecture & Roles](#2-architecture--roles)
3. [Prerequisites](#3-prerequisites)
4. [Step-by-Step Setup Checklist](#4-step-by-step-setup-checklist)
5. [Directory Structure & Setup](#5-directory-structure--setup)
6. [Configuration File](#6-configuration-file)
7. [The 6-Phase Pipeline](#7-the-6-phase-pipeline)
8. [State Management](#8-state-management)
9. [Prompt Files (Full Content)](#9-prompt-files-full-content)
10. [Schema Files (Full Content)](#10-schema-files-full-content)
11. [Verdict System & Iteration Logic](#11-verdict-system--iteration-logic)
12. [JSONL Parsing Reference (Codex CLI Output Format)](#12-jsonl-parsing-reference-codex-cli-output-format)
13. [Error Handling & Escalation](#13-error-handling--escalation)
14. [Notification Wiring (OpenClaw message tool)](#14-notification-wiring-openclaw-message-tool)
15. [AGENTS.md & HEARTBEAT.md Integration](#15-agentsmd--heartbeatmd-integration)
16. [Heartbeat Recovery Integration](#16-heartbeat-recovery-integration)
17. [Concurrency (Multiple Features)](#17-concurrency-multiple-features)
18. [Operational Runbook Template (PIPELINE-SPEC.md)](#18-operational-runbook-template-pipeline-specmd)
19. [Example Working Doc](#19-example-working-doc)
20. [Step-by-Step Execution Checklist (Per Feature)](#20-step-by-step-execution-checklist-per-feature)
21. [Lessons Learned & Gotchas](#21-lessons-learned--gotchas)
22. [Adapting to Your Project](#22-adapting-to-your-project)

---

## 1. How It Works (Big Picture)

```
Human writes a "Working Doc" (plain English feature description)
        ↓
  [Phase 1] Setup — branch created, state initialized
        ↓
  [Phase 2] Spec Generation — coding agent reads codebase + working doc → produces implementation spec
        ↓
  [Phase 3] Spec Audit Loop — coding agent audits the spec (up to N iterations, fixes between rounds)
        ↓
  [Phase 4] Implementation — coding agent implements the spec (writes actual code)
        ↓
  [Phase 4.5] Build Verification — run build + lint
        ↓
  [Phase 5] Post-Impl Audit Loop — coding agent audits the code (up to N iterations, fixes between rounds)
        ↓
  [Phase 6] Finalize — git commit, push, summary report
        ↓
  Human reviews the feature branch and merges
```

The key insight: **OpenClaw's AI is the orchestrator/PM, not the coder.** It manages phases, parses structured output, tracks state, and escalates. A separate coding agent (Codex CLI) does the actual code reading/writing. This separation prevents the orchestrator from hallucinating code and gives the coding agent full sandbox access to the repo.

---

## 2. Architecture & Roles

### The Orchestrator (OpenClaw's AI — you)
- Reads the pipeline runbook at the start of every run
- Manages phase transitions (never skip, never improvise)
- Launches coding agent sessions with assembled prompts
- Parses structured JSON verdicts from audit phases
- Extracts issue ledgers and feeds them back for revisions
- Tracks state in JSON files
- Escalates to the human when things go wrong
- Sends progress notifications via OpenClaw's `message` tool

### The Coding Agent (Codex CLI, or equivalent)
- Has read/write access to the repo
- Reads the codebase to understand existing patterns
- Writes implementation specs
- Audits specs and implementations
- Writes actual code
- Runs build/lint/test commands
- Outputs structured JSON when given a schema

### The Human
- Writes the working doc (input)
- Reviews the feature branch (output)
- Handles escalations
- Merges to staging/main

### Separation of Concerns
```
┌─────────────────────────────────────────────┐
│  Human                                       │
│  - Writes working doc                        │
│  - Reviews output                            │
│  - Handles escalations                       │
│  - Merges branches                           │
└──────────┬──────────────────────────────────┘
           │ working doc
           ▼
┌─────────────────────────────────────────────┐
│  OpenClaw AI (Orchestrator / PM)             │
│  - Phase management                          │
│  - Prompt assembly                           │
│  - Verdict parsing                           │
│  - Issue ledger extraction                   │
│  - State tracking                            │
│  - Escalation                                │
│  - Notifications                             │
└──────────┬──────────────────────────────────┘
           │ codex exec / exec resume (via shell)
           ▼
┌─────────────────────────────────────────────┐
│  Coding Agent (Codex CLI)                    │
│  - Reads codebase                            │
│  - Writes specs                              │
│  - Audits specs/code                         │
│  - Writes implementation code                │
│  - Runs build/test commands                  │
│  - Outputs structured JSON                   │
└─────────────────────────────────────────────┘
```

---

## 3. Prerequisites

### Required Software
- **OpenClaw** — running on your machine with an AI model configured
- **Codex CLI** — OpenAI's coding agent CLI (`codex exec`, `codex exec resume`). Install via `npm install -g @openai/codex` or similar. Note the binary path after install.
- **Git** — for branch management
- **Your project's build toolchain** — Node.js/npm, Python, Rust, Go, whatever. Needed for build/lint verification.
- **jq** — for parsing JSON output from the coding agent (`brew install jq` / `apt install jq`)

### Required Configuration
- A Git repo with a base branch (e.g., `staging`, `develop`, `main`)
- Codex CLI authenticated with a model that can read/write code (run `codex auth` or equivalent)
- OpenClaw configured with at least one notification channel (webchat, Discord, Telegram, etc.)

### If NOT Using Codex CLI
You can adapt this to any coding agent that supports:
1. Running with a prompt against a repo directory
2. Outputting structured JSON (or text that can be parsed)
3. Session resume (continuing a previous conversation thread)
4. An autonomous/full-auto mode

Candidates: Claude Code (`claude`), Aider, OpenCode, etc. Adapt the shell commands but keep the pipeline logic identical.

---

## 4. Step-by-Step Setup Checklist

Follow this in order. Each step must be complete before moving on.

### Phase A: Install Dependencies
- [ ] **A1.** Verify Git is installed: `git --version`
- [ ] **A2.** Verify jq is installed: `jq --version` (if missing: `brew install jq` or `apt install jq`)
- [ ] **A3.** Install Codex CLI: `npm install -g @openai/codex` (or your coding agent)
- [ ] **A4.** Note the Codex binary path: `which codex` → save this for config
- [ ] **A5.** Authenticate Codex: `codex auth` (or equivalent for your agent)
- [ ] **A6.** Verify Codex works: `codex exec -m <your-model> "echo hello"` — should run without errors
- [ ] **A7.** Verify your project builds: `cd <repo> && <build_command>` (e.g., `npm run build`)

### Phase B: Create Pipeline Directory Structure
- [ ] **B1.** Choose your pipeline directory (recommended: `~/.openclaw/workspace/pipeline/`)
- [ ] **B2.** Run: `mkdir -p <pipeline_dir>/{prompts,schemas,state,logs,inbox}`
- [ ] **B3.** Create `config.yaml` in the pipeline dir — fill in ALL paths (see Section 6)

### Phase C: Create Prompt Files
- [ ] **C1.** Create `prompts/prompt0-working-doc.md` — copy from Section 9.1
- [ ] **C2.** Create `prompts/prompt1-spec-writer.md` — copy from Section 9.2
- [ ] **C3.** Create `prompts/prompt2-spec-auditor.md` — copy from Section 9.3
- [ ] **C4.** Create `prompts/prompt3-post-audit.md` — copy from Section 9.4
- [ ] **C5.** Create `prompts/implementation-primer.md` — copy from Section 9.5
- [ ] **C6.** **IMPORTANT:** Edit `implementation-primer.md` to match YOUR project's environment constraints (build commands, port restrictions, tool configs, branch names)

### Phase D: Create Schema Files
- [ ] **D1.** Create `schemas/spec-audit-verdict.json` — copy from Section 10.1
- [ ] **D2.** Create `schemas/impl-audit-verdict.json` — copy from Section 10.2

### Phase E: Create the Operational Runbook
- [ ] **E1.** Create `PIPELINE-SPEC.md` in your pipeline dir — use the template from Section 18
- [ ] **E2.** Replace ALL `$VARIABLE` placeholders with your actual values from config.yaml
- [ ] **E3.** Review every shell command in the runbook and verify paths are correct

### Phase F: Wire Into OpenClaw
- [ ] **F1.** Add pipeline instructions to your `AGENTS.md` (see Section 15.1)
- [ ] **F2.** Add heartbeat recovery to your `HEARTBEAT.md` (see Section 16)
- [ ] **F3.** Verify your notification channel works: test-send a message using the `message` tool (see Section 14)

### Phase G: Dry Run
- [ ] **G1.** Write a small test working doc (see Section 19 for an example)
- [ ] **G2.** Tell your OpenClaw: "Run the implementation pipeline on this working doc: <path>"
- [ ] **G3.** Monitor the first run end-to-end — expect to find environment-specific issues
- [ ] **G4.** After the first run, update `implementation-primer.md` and `PIPELINE-SPEC.md` with any environment constraints you discovered
- [ ] **G5.** Document lessons in your memory files

---

## 5. Directory Structure & Setup

```
<your-workspace>/pipeline/
├── PIPELINE-SPEC.md          # Your operational runbook (generated from Section 18 template)
├── config.yaml               # Paths and settings
├── prompts/
│   ├── prompt0-working-doc.md    # Interactive working doc creation helper
│   ├── prompt1-spec-writer.md    # Spec generation prompt
│   ├── prompt2-spec-auditor.md   # Spec audit prompt
│   ├── prompt3-post-audit.md     # Implementation audit prompt
│   └── implementation-primer.md  # Prepended to spec for implementation phase
├── schemas/
│   ├── spec-audit-verdict.json   # Structured output schema for spec audits
│   └── impl-audit-verdict.json   # Structured output schema for impl audits
├── state/                        # Auto-created per feature
│   └── <feature-slug>/
│       ├── state.json
│       ├── working-doc.md
│       ├── spec.md
│       ├── audit-N.json
│       ├── post-audit-N.json
│       ├── context-brief.md
│       └── ... (all artifacts)
├── logs/                         # General logs
└── inbox/                        # Drop working docs here to trigger pipeline
```

---

## 6. Configuration File

Create `pipeline/config.yaml`:

```yaml
# Implementation Pipeline — Configuration
# ⚠️ Adapt ALL paths to your environment before using

repo_path: /path/to/your/repo                    # Git repo root (absolute path)
codex_bin: ~/.local/bin/codex                     # Path to coding agent binary
model: gpt-5.3-codex                             # Model for coding agent sessions
max_spec_iterations: 3                            # Max spec audit rounds before escalate
max_impl_iterations: 3                            # Max impl audit rounds before escalate
state_dir: ~/.openclaw/workspace/pipeline/state/
prompts_dir: ~/.openclaw/workspace/pipeline/prompts/
schemas_dir: ~/.openclaw/workspace/pipeline/schemas/
logs_dir: ~/.openclaw/workspace/pipeline/logs/
inbox_dir: ~/.openclaw/workspace/pipeline/inbox/
notification_channel: webchat                     # webchat | discord | telegram | signal | slack
base_branch: staging                              # Branch that features are based off
branch_prefix: feature/                           # Prefix for feature branches
build_command: "npm run build"                    # Your project's build command
lint_command: "npm run lint"                      # Your project's lint command
test_command: "npm test"                          # Your project's test command (optional)
```

---

## 7. The 6-Phase Pipeline

### Phase 1: Setup

**What happens:** Create a feature branch, initialize state tracking, copy the working doc.

**Commands (run via exec tool):**
```bash
cd "$REPO_PATH"
git fetch origin
git checkout $BASE_BRANCH
git pull origin $BASE_BRANCH
BRANCH="${BRANCH_PREFIX}${SLUG}"
git checkout -b "$BRANCH" "$BASE_BRANCH"

mkdir -p "$STATE_DIR/$SLUG"
cp "$WORKING_DOC" "$STATE_DIR/$SLUG/working-doc.md"
# Initialize state.json (see Section 8)
```

**State update:** `phase: "setup"`, `status: "in_progress"`
**Notification:** Send via message tool (see Section 14)

### Slug Derivation
From the working doc filename: `signal-feed-working-doc.md` → `signal-feed`
Or from the first H1 heading, kebab-cased.
If collision, append timestamp: `signal-feed-20260216`

---

### Phase 2: Spec Generation

**What happens:** Coding agent reads the codebase + working doc and produces a detailed implementation spec.

**Command (run via exec tool — this is a long-running process):**
```bash
JSONL_LOG="$STATE_DIR/$SLUG/spec-gen.jsonl"

$CODEX_BIN exec \
  -m "$MODEL" \
  -C "$REPO_PATH" \
  --json \
  --full-auto \
  -o "$STATE_DIR/$SLUG/spec.md" \
  "$(cat $PROMPTS_DIR/prompt1-spec-writer.md)

---
## WORKING DOC
$(cat $STATE_DIR/$SLUG/working-doc.md)
---

Read the codebase first (especially any project conventions files like CLAUDE.md, AGENTS.md, CONTRIBUTING.md).
Generate the implementation spec.
Your FINAL message must be the complete spec content — nothing else." \
  | tee "$JSONL_LOG"
```

**Extract thread_id for session continuity (see Section 12 for JSONL format):**
```bash
SPEC_THREAD_ID=$(head -1 "$JSONL_LOG" | jq -r '.thread_id')
```

**State update:** `spec_thread_id`, `phase: "spec_gen"`

---

### Phase 3: Spec Audit Loop

**What happens:** A separate coding agent session audits the spec against the working doc and codebase. If issues are found, the spec is revised and re-audited. Up to `max_spec_iterations` rounds.

**Critical rules:**
1. The auditor runs in ONE continuous thread across all iterations. Iteration 1 starts the thread; iterations 2+ resume it.
2. After every audit, extract an issue ledger file.
3. Feed the ledger to both the revision step AND the re-audit step.

#### Iteration 1 (First Audit)
```bash
AUDIT_LOG="$STATE_DIR/$SLUG/audit-1.jsonl"

$CODEX_BIN exec \
  -m "$MODEL" \
  -C "$REPO_PATH" \
  --json \
  --full-auto \
  --output-schema "$SCHEMAS_DIR/spec-audit-verdict.json" \
  -o "$STATE_DIR/$SLUG/audit-1.json" \
  "$(cat $PROMPTS_DIR/prompt2-spec-auditor.md)

---
## WORKING DOC
$(cat $STATE_DIR/$SLUG/working-doc.md)
---
## SPEC TO AUDIT
$(cat $STATE_DIR/$SLUG/spec.md)
---

Audit this spec against the working doc and the actual codebase.
Read relevant source files to verify assumptions.
Output your verdict as structured JSON matching the provided schema.
Assign each issue a stable ID (SPEC-1, SPEC-2, etc.) that will persist across re-audits." \
  | tee "$AUDIT_LOG"
```

**Save audit thread ID:**
```bash
SPEC_AUDIT_THREAD_ID=$(head -1 "$AUDIT_LOG" | jq -r '.thread_id')
```

#### After Each Audit — Extract Issue Ledger
```bash
jq -r '.issues[] | "- [\(.id)] [\(.severity)] [\(.status)] \(.section): \(.issue) → Fix: \(.suggested_fix)"' \
  "$STATE_DIR/$SLUG/audit-$N.json" > "$STATE_DIR/$SLUG/spec-issues-$N.md"
```

#### Parse Verdict
```bash
VERDICT=$(jq -r '.verdict' "$STATE_DIR/$SLUG/audit-$N.json")
BLOCKER_COUNT=$(jq -r '.blocker_count' "$STATE_DIR/$SLUG/audit-$N.json")
```

#### Verdict Handling

| Verdict | Action |
|---------|--------|
| `IMPLEMENTABLE` | Proceed to Phase 4 |
| `IMPLEMENTABLE_WITH_NOTES` | Proceed to Phase 4 (minor/info logged) |
| `NOT_IMPLEMENTABLE_YET` | Revise spec → re-audit |
| `ESCALATE` | Pause, notify human |

#### Spec Revision (When NOT_IMPLEMENTABLE_YET)
```bash
# Filter to unresolved/new blockers only
FIXES=$(jq -r '.issues[] | select(.severity == "critical" or .severity == "major") | select(.status != "resolved") | "- [\(.id)] [\(.severity)] \(.section): \(.issue) → Fix: \(.suggested_fix)"' \
  "$STATE_DIR/$SLUG/audit-$N.json")

$CODEX_BIN exec resume "$SPEC_THREAD_ID" \
  --json \
  --full-auto \
  "The spec auditor found issues that must be fixed. Address ONLY the following issues — do not change anything else.

ISSUE LEDGER (fix these):
$FIXES

CRITICAL: Return a FULLY UPDATED END-TO-END SPEC — not a patch, not a diff. The complete revised spec.

Write the complete revised spec to: $STATE_DIR/$SLUG/spec.md

Use a shell command like:
cat > '$STATE_DIR/$SLUG/spec.md' << 'SPECEOF'
...complete spec...
SPECEOF" \
  | tee "$STATE_DIR/$SLUG/spec-fix-$N.jsonl"
```

#### Iteration 2+ (Re-Audit via Resume)
```bash
$CODEX_BIN exec resume "$SPEC_AUDIT_THREAD_ID" \
  --json \
  --full-auto \
  "The spec has been revised. Re-audit it now.

PREVIOUS ISSUE LEDGER:
$(cat $STATE_DIR/$SLUG/spec-issues-$((N-1)).md)

INSTRUCTIONS:
1. For each previous issue, verify if resolved. Mark 'resolved' or 'unresolved'.
2. Only flag NEW issues if INTRODUCED BY THE REVISION. Do not expand scope.
3. New issues get status 'new' and fresh IDs continuing the sequence.
4. Read the updated spec from: $STATE_DIR/$SLUG/spec.md
5. Write your structured JSON verdict to: $STATE_DIR/$SLUG/audit-$N.json

The JSON must match this schema:
$(cat $SCHEMAS_DIR/spec-audit-verdict.json)

Write the JSON verdict file using a shell command." \
  | tee "$STATE_DIR/$SLUG/audit-$N.jsonl"
```

**⚠️ `exec resume` limitations:** Does NOT support `-o` or `--output-schema` flags. That's why we include the schema inline and tell the agent to write the file via shell command. See Section 12 for details.

#### Max Iteration Behavior
When `max_spec_iterations` is reached:
- **blocker_count = 0** → Auto-promote to `IMPLEMENTABLE_WITH_NOTES`, proceed
- **blocker_count > 0** → `ESCALATE` to human. Do NOT proceed.

---

### Phase 4: Implementation

**What happens:** Coding agent implements the spec — writes actual code in the repo.

```bash
IMPL_LOG="$STATE_DIR/$SLUG/impl.jsonl"

$CODEX_BIN exec \
  -m "$MODEL" \
  -C "$REPO_PATH" \
  --json \
  --full-auto \
  "$(cat $PROMPTS_DIR/implementation-primer.md)

---
## SPEC TO IMPLEMENT
$(cat $STATE_DIR/$SLUG/spec.md)
---

Read project conventions before writing any code.
Implement this spec end-to-end.
Run '$BUILD_COMMAND' and '$LINT_COMMAND' after implementation to verify.
Report what files you created or modified as your final message." \
  | tee "$IMPL_LOG"
```

**Extract thread_id:**
```bash
IMPL_THREAD_ID=$(head -1 "$IMPL_LOG" | jq -r '.thread_id')
```

**State update:** `impl_thread_id`, `phase: "implementation"`

#### Codebase Integrity Safeguards
The implementation primer tells the coding agent to **halt and report mismatches** if the spec references files/modules/tables that don't exist. If it reports mismatches:
1. Set `phase: "needs_human"`, `status: "needs_human"`
2. Log the mismatch in `state/<slug>/mismatch-report.md`
3. Notify the human — the spec was written against incorrect assumptions

---

### Phase 4.5: Build Verification

**What happens:** Run build and lint to verify the implementation compiles.

```bash
cd "$REPO_PATH"
$BUILD_COMMAND > "$STATE_DIR/$SLUG/build-log.txt" 2>&1
BUILD_EXIT=$?

$LINT_COMMAND > "$STATE_DIR/$SLUG/lint-log.txt" 2>&1
LINT_EXIT=$?
```

**If either fails:** Resume the impl session with the errors:
```bash
$CODEX_BIN exec resume "$IMPL_THREAD_ID" \
  --json \
  --full-auto \
  "Build/lint failed. Fix these errors:

BUILD OUTPUT:
$(cat $STATE_DIR/$SLUG/build-log.txt)

LINT OUTPUT:
$(cat $STATE_DIR/$SLUG/lint-log.txt)

Fix all errors and re-run build and lint to verify." \
  | tee "$STATE_DIR/$SLUG/build-fix.jsonl"
```

This counts toward `impl_iteration`.

**State update:** `build_passed`, `lint_passed`, `phase: "build_verify"`

---

### Phase 5: Post-Implementation Audit Loop

**What happens:** A separate coding agent session audits the actual code against the spec. Same iteration logic as Phase 3 but for code.

**Same session continuity rule:** One continuous thread across all iterations.

**Get the diff:**
```bash
cd "$REPO_PATH"
DIFF=$(git diff $BASE_BRANCH..HEAD)
```

#### Iteration 1 (First Audit)
```bash
POSTAUDIT_LOG="$STATE_DIR/$SLUG/post-audit-1.jsonl"

$CODEX_BIN exec \
  -m "$MODEL" \
  -C "$REPO_PATH" \
  --json \
  --full-auto \
  --output-schema "$SCHEMAS_DIR/impl-audit-verdict.json" \
  -o "$STATE_DIR/$SLUG/post-audit-1.json" \
  "$(cat $PROMPTS_DIR/prompt3-post-audit.md)

---
## SPEC
$(cat $STATE_DIR/$SLUG/spec.md)
---
## IMPLEMENTATION (Git Diff)
$DIFF
---

Audit this implementation against the spec.
Read the actual source files for full context.
Hunt for bugs, missing error handling, and spec deviations.
Assign each issue a stable ID (IMPL-C1 for compliance, IMPL-B1 for bugs).
Output your verdict as structured JSON matching the provided schema." \
  | tee "$POSTAUDIT_LOG"
```

**Save audit thread ID:**
```bash
IMPL_AUDIT_THREAD_ID=$(head -1 "$POSTAUDIT_LOG" | jq -r '.thread_id')
```

#### After Each Audit — Extract Issue Ledger
```bash
jq -r '
  ([.compliance_issues[]? | "- [\(.id)] [\(.severity)] [\(.status)] COMPLIANCE \(.spec_section): \(.issue) → Fix: \(.fix)"] +
   [.bugs[]? | "- [\(.id)] [\(.severity)] [\(.status)] BUG \(.file // "unknown"): \(.description) → Fix: \(.fix)"])
  | join("\n")
' "$STATE_DIR/$SLUG/post-audit-$N.json" > "$STATE_DIR/$SLUG/impl-issues-$N.md"
```

#### Fix Code (When NOT_COMPLIANT)
```bash
FIXES=$(jq -r '
  ([.compliance_issues[]? | select(.severity == "critical" or .severity == "major") | select(.status != "resolved") | "- [\(.id)] [\(.severity)] COMPLIANCE \(.spec_section): \(.issue) → Fix: \(.fix)"] +
   [.bugs[]? | select(.severity == "critical" or .severity == "major") | select(.status != "resolved") | "- [\(.id)] [\(.severity)] BUG \(.file // "unknown"): \(.description) → Fix: \(.fix)"])
  | join("\n")
' "$STATE_DIR/$SLUG/post-audit-$N.json")

$CODEX_BIN exec resume "$IMPL_THREAD_ID" \
  --json \
  --full-auto \
  "The post-implementation audit found issues. Apply ONLY these fixes:

ISSUE LEDGER (fix these):
$FIXES

After fixing, run '$BUILD_COMMAND' and '$LINT_COMMAND' to verify." \
  | tee "$STATE_DIR/$SLUG/fix-$N.jsonl"
```

#### Iteration 2+ (Re-Audit via Resume)
```bash
POSTAUDIT_LOG="$STATE_DIR/$SLUG/post-audit-$N.jsonl"
DIFF=$(git diff $BASE_BRANCH..HEAD)

$CODEX_BIN exec resume "$IMPL_AUDIT_THREAD_ID" \
  --json \
  --full-auto \
  "The implementation has been updated to address your previous findings. Re-audit it now.

PREVIOUS ISSUE LEDGER:
$(cat $STATE_DIR/$SLUG/impl-issues-$((N-1)).md)

UPDATED DIFF:
$DIFF

INSTRUCTIONS:
1. For each issue in the previous ledger, verify if it's resolved. Mark as 'resolved' or 'unresolved'.
2. Only flag NEW issues if INTRODUCED BY THE FIX. Do not expand scope.
3. New issues get status 'new' and a fresh ID continuing the sequence.
4. Read the actual source files for full context.
5. Write your structured JSON verdict to: $STATE_DIR/$SLUG/post-audit-$N.json

The JSON must match this schema:
$(cat $SCHEMAS_DIR/impl-audit-verdict.json)

Write the JSON verdict file using a shell command." \
  | tee "$POSTAUDIT_LOG"
```

#### Max Iteration Behavior
Same as Phase 3: blocker_count = 0 → auto-promote to `COMPLIANT_WITH_NOTES`; blocker_count > 0 → ESCALATE.

---

### Phase 6: Finalize

**What happens:** Commit, push, generate summary, notify human.

```bash
cd "$REPO_PATH"

git add -A -- \
  ':(exclude).env*' \
  ':(exclude)*.log' \
  ':(exclude)node_modules' \
  ':(exclude).DS_Store'

git commit -m "feat($SLUG): implement feature from working doc

Spec passed audit in $SPEC_ITERATIONS iteration(s).
Implementation passed audit in $IMPL_ITERATIONS iteration(s).
Automated by Implementation Pipeline."

git push origin "$BRANCH"
```

**Generate summary** in `state/<slug>/summary.md`:
- Branch name, duration
- Spec iterations / max, impl iterations / max
- Final verdicts
- Files changed (`git diff --name-only $BASE_BRANCH..HEAD`)
- Warnings (if max iterations hit)
- Next steps (review, test, merge)

**Send notification immediately** via message tool — don't wait for heartbeat.

**State update:** `phase: "completed"`, `status: "completed"`

---

## 8. State Management

Each feature gets: `state/<feature-slug>/state.json`

```json
{
  "feature_slug": "<slug>",
  "working_doc_path": "state/<slug>/working-doc.md",
  "branch": "feature/<slug>",
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
  "started_at": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>",
  "status": "pending|in_progress|completed|failed|needs_human",
  "error": null,
  "resume_hint": null
}
```

### `resume_hint` Field
**Update this EVERY time you update `phase` or `status`.** One sentence: what just happened, what to do next. This is the cold-resume orientation — a fresh session reads this and knows immediately how to proceed without re-parsing all artifacts.

Example: `"Spec audit iteration 2 completed with NOT_IMPLEMENTABLE_YET, blocker_count=1. Need to revise spec and re-audit (iteration 3)."`

### Artifacts Per Feature
```
state/<slug>/
├── state.json            # Pipeline state (the source of truth)
├── working-doc.md        # Input working doc (never modified)
├── spec.md               # Generated spec (updated each revision)
├── spec-gen.jsonl        # Raw JSONL from Phase 2
├── audit-N.json          # Structured audit verdict (iteration N)
├── audit-N.jsonl         # Raw JSONL from audit
├── spec-issues-N.md      # Issue ledger extracted from audit N
├── spec-fix-N.jsonl      # Raw JSONL from spec revision
├── impl.jsonl            # Raw JSONL from Phase 4
├── build-log.txt         # Build output
├── lint-log.txt          # Lint output
├── post-audit-N.json     # Structured impl audit (iteration N)
├── post-audit-N.jsonl    # Raw JSONL from impl audit
├── impl-issues-N.md      # Issue ledger from impl audit N
├── fix-N.jsonl           # Raw JSONL from impl fixes
├── context-brief.md      # Living summary for session continuity
├── mismatch-report.md    # If codebase integrity check fails
└── summary.md            # Final summary report
```

### Context Brief Generation
Between major phases (after spec audit passes, after implementation, after impl audit passes), generate/update `context-brief.md`:

```markdown
# Context Brief: <feature-slug>
## Current Phase: <phase>
## Branch: <branch>
## Last Updated: <timestamp>

## What's Done
- <summary of completed work>

## What's Known
- <key decisions, constraints, environment notes>

## What to Skip
- <things already tried that don't work>

## Open Issues
- <residual items from latest audit, if any>
```

This prevents new/resumed sessions from re-discovering known constraints. It saves tokens and time.

---

## 9. Prompt Files (Full Content)

### 9.1 `prompts/prompt0-working-doc.md` — Interactive Working Doc Creator

This is an **optional** helper. You give this prompt to a coding agent interactively (not as part of the pipeline), and it helps the human think through and produce a structured working doc. It's the "pre-pipeline" phase.

```markdown
You are a senior engineering partner helping me create a working doc for a new feature. Your job is to help me think through the implementation clearly and completely, then produce a structured document that can be handed to a spec writer.

## Important Context

The working doc you help me produce will be used as the source of truth for an implementation spec. That spec will be handed to an LLM for implementation. This means:
- Everything must be precise enough that a spec writer won't have to guess
- Design decisions must be made HERE, not deferred to the spec
- The "spirit" of the feature must be captured
- Current codebase reality must be accounted for

## Critical: The Alignment Problem

This codebase is complex. The most dangerous mistakes happen when we THINK we understand how something works, but we're wrong.

To prevent this:
- **Never assume you understand how two systems connect. Verify by reading the code.**
- **When asked "how does X work?" — read the actual code, don't answer from memory.**
- **If you're not sure, say so.** "I'd need to look at [file] to confirm" is always better than a confident wrong answer.
- **Surface hidden connections proactively.**

## Phases

### PHASE 1: Understand My Goal
Ask what I want to build and why. Restate to confirm. Ask clarifying questions. Output: approved Goal Statement.

### PHASE 2: Codebase Discovery
- Map what exists (files, functions, tables, dependencies)
- Map how things connect (upstream/downstream, data flow)
- Identify the surgery zone (what we touch, what's load-bearing, what's safe)
- Output: Current State Map with dependency graph

### PHASE 3: Impact Analysis
For each piece of code in the surgery zone:
- Direct impact, upstream callers, downstream calls
- Data shape changes, side effect risks
- Confidence level (HIGH/MEDIUM/LOW)

### PHASE 4: Design Decisions
One decision at a time. Present options grounded in actual codebase. Recommend, but let me decide. Update impact map after each decision.

### PHASE 5: Draft the Working Doc
Synthesize into structured doc:
- Goal, Context, Current State, Impact Analysis
- Design Decisions, How It Should Work
- Technical Approach, What NOT to Change
- Scope, Edge Cases & Open Questions

## Rules
- Do not design in a vacuum — check the code
- Do not make decisions for me — present options
- Do not over-engineer — simplest approach wins
- Do not scope-creep — build exactly what's asked
- Be honest about complexity and uncertainty
- Show evidence when challenged
- Surface risks proactively
```

---

### 9.2 `prompts/prompt1-spec-writer.md` — Spec Generation

This is the core spec-writing prompt. It tells the coding agent exactly what structure and quality level the spec must have. Every section exists to constrain a specific LLM failure mode.

```markdown
You are an engineering spec writer for an existing, complex repo. Your job is to produce an implementation spec so precise that an LLM can execute it with minimal bugs, no drift, and no bloat.

## Why This Matters

The spec you produce will be handed to an LLM for implementation. LLMs implementing code tend to:
- Drift from the spec (adding unrequested features, "improving" architecture)
- Leave implementations incomplete (80% done, stubs, TODOs)
- Invent patterns instead of matching existing ones
- Create unnecessary files, abstractions, and utilities
- Swallow errors silently
- Skip edge cases
- Not verify their own work

Your spec must structurally prevent all of these. Every section exists to constrain a specific failure mode.

---

## Operating Principles (Restate these verbatim in Section 0 of every spec)

1) **Faithfulness to the working doc**: The working doc is the source of truth for WHAT to build and WHY. Do not reinterpret, optimize, or extend beyond what it describes. If something is ambiguous, mark it as "REQUIRES CLARIFICATION" — do not fill in the gap with assumptions.

2) **Minimal diff, maximal clarity**: Prefer surgical changes to existing files over creating new files. Only create a new file if the responsibility genuinely doesn't belong anywhere that already exists. Only refactor if the working doc explicitly calls for it.

3) **Build the target state directly**: Do not maintain backward compatibility with code being replaced. Do not add feature flags to gate new functionality. Do not create "old path / new path" conditional logic. If something is being replaced, replace it. The old code gets removed in the same step the new code goes in.

4) **Match existing patterns**: Before writing any new code, reference how similar things are already done in the repo. New code must match existing conventions for naming, error handling, file organization, and import style. Cite specific existing examples in the spec.

5) **Comment like the reader has zero context**: Every function gets a docstring explaining what it does and why it exists. Every non-obvious code block gets an inline comment explaining the intent. Comments explain business logic and "why," not just "what." Assume the person reading the code has never seen this codebase and doesn't have access to this spec.

6) **Complete implementations only**: No stubs. No placeholder functions. No TODO comments. No "will be implemented in a future step." Every piece of code in the spec must be fully functional when implemented. If something is out of scope, say so explicitly in the spec and do not include skeleton code for it.

7) **Verify your own work**: Every implementation step includes a concrete verification check. The implementer must be able to prove each step works before moving to the next.

---

## Spec Structure (Required Sections)

### 0) Implementation Contract
- Restate the Operating Principles above verbatim
- Define any terms specific to this feature
- List the explicit scope boundary: "This spec covers X. It does NOT cover Y."
- **Negative constraints**: List specific things the implementer must NOT do

### 1) Current State
- How the relevant parts of the system work today
- Exact files and functions involved (with file paths)
- Exact DB tables and columns referenced
- Known issues or gaps the working doc is addressing
- **Existing patterns to match**: Show 1-2 concrete examples of how the repo already handles similar functionality

### 2) Target State
- How it should work after implementation
- Text-based architecture/data flow diagram
- Explicit delta: what changes, what stays the same, what gets removed
- If replacing existing behavior: before/after comparison

### 3) Migration Narrative
Write this as a clear "Before → After" story:
- "Before: [how the relevant flow works today]"
- "After: [how it works once this spec is implemented]"
- If behavior has states (loading, error, empty, populated), define each state and what the UI/API returns in each
- If data flows through multiple systems, trace the full path before and after

### 4) Data Contracts
For each new or modified payload:
- **Canonical JSON shape** with types and example values
- **Required vs optional fields**
- **Identity keys**: what makes a record unique, how duplicates are detected
- **Timestamp semantics**: what gets set when, what's immutable after creation
- **Validation rules**: min/max, allowed values, format constraints

### 5) Database Migrations
- Full SQL for any new tables, columns, indexes, triggers, RLS policies, RPCs
- Migration ordering if multiple migrations are needed
- Note which migrations are destructive vs. additive
- For each table: explain its purpose in a SQL comment
- **If your project has no database, replace this section with the relevant infrastructure section (API contracts, config files, etc.)**

### 6) Implementation Steps
An ordered sequence of steps. Each step must include:
- **What**: One-sentence description of the change
- **Files**: Exact file paths that get modified or created
- **Details**: What specifically changes in each file
- **Existing pattern reference**: "Follow the pattern in [existing_file:function_name]"
- **Tests**: Which tests from the Test Plan (Section 6.5) should be written in this step
- **Done when**: A concrete, testable assertion
- **Do not**: Step-specific things the implementer must avoid

### 6.5) Test Plan
For each major new function or endpoint, define test scenarios with:
- Short descriptive name
- Function/Endpoint being tested
- Setup (preconditions)
- Input (exact values)
- Expected output (exact return value or behavior)
- Why this matters (what bug this test prevents)

Required categories: happy path, empty input, error case, boundary case.

### 7) Concrete Patch List
For every file that changes:
- FILE: exact path
- ACTION: MODIFY | CREATE | DELETE
- CHANGES: function/class/block + what changes and why
- PLACEMENT: where new code goes in existing files
- PATTERN REF: existing file:function to match style of

### 8) Code Quality Checklist
- Comment requirements
- Error handling specifics
- Logging requirements
- Naming conventions for new code
- Test coverage requirements

### 9) Verification Plan
For each implementation step:
- Manual test
- Log check
- DB check
- Error case verification
- Automated test reference

### 10) Known Risks & Open Questions
- Assumptions (confirmed vs requires confirmation)
- Edge cases and handling
- Things that could break

---

## Scorecard (Must be 16/16 to pass)

| # | Category | Check |
|---|----------|-------|
| 1 | Faithfulness | Spec implements exactly what the working doc describes |
| 2 | Contracts | Every payload has a canonical JSON shape with types |
| 3 | Contracts | Every DB change has full SQL with comments |
| 4 | Completeness | Every step has a concrete "done when" assertion |
| 5 | Completeness | Every file in patch list has specific changes listed |
| 6 | Existing Patterns | At least one pattern reference per major component |
| 7 | Negative Constraints | Section 0 includes feature-specific "do NOT" list |
| 8 | Code Quality | All new names explicitly specified |
| 9 | Code Quality | Comment requirements specified per function |
| 10 | Verification | Each step has testable verification |
| 11 | Minimalism | No feature flags, no old/new paths, no unnecessary files |
| 12 | Scope Boundary | Explicit in/out of scope |
| 13 | Current State | Exact existing files/functions identified |
| 14 | Before → After | Behavior changes shown concretely |
| 15 | Testing | Test scenarios with exact inputs/outputs |
| 16 | Testing | Tests cover happy path, empty, error, boundary |
```

---

### 9.3 `prompts/prompt2-spec-auditor.md` — Spec Audit

```markdown
You are a senior staff engineer and spec auditor. Your job is to review an implementation spec and produce structured feedback that the spec-writing LLM can use to fix every issue.

## Your Primary Deliverable

Your output is NOT a report for a human. It is a **revision instruction set** for the LLM that wrote the spec. Every issue you identify must include the exact fix. "This is vague" is not useful. "Rewrite this line to say [exact text]" is useful.

## Scope Boundary (CRITICAL)

**Audit ONLY against the working doc requirements.** The working doc defines what is in scope. Do not:
- Flag issues for features the working doc didn't request
- Expand scope to adjacent concerns (e.g., if the working doc says "use Supabase OAuth," don't audit PKCE implementation details that Supabase handles internally)
- Raise issues about systems or components not touched by this feature
- Treat "could be better" as a finding unless it creates a concrete bug or spec gap

If something feels like it *should* be in scope but the working doc doesn't mention it, flag it as severity `info` with a note that it's out of scope per the working doc.

## Re-Audit Protocol (For Iterations 2+)

If you are provided a **previous issue ledger**, you MUST:
1. **Go through each previous issue by ID** and verify if it's resolved in the current spec
2. Mark each as `resolved` or `unresolved` in your output
3. **Only flag NEW issues** if they were **introduced by the revision** — do not expand scope
4. New issues get status `new` and a fresh ID continuing the sequence
5. Do NOT re-raise issues that were already resolved
6. Your verdict should reflect the current state: if all blockers are resolved, the spec is implementable even if minor/info items remain

## Why This Audit Exists

Implementation specs get handed to LLMs that will:
- Build things the working doc didn't ask for (if the spec drifted)
- Guess when the spec is ambiguous (and guess wrong)
- Skip things the spec left vague ("handle errors appropriately" → empty try/except)
- Invent patterns instead of following existing ones (if no examples were cited)
- Leave implementations incomplete (if "done" isn't concretely defined)
- Touch files outside the spec's scope (if boundaries aren't explicit)

Your audit must catch every place this can happen, and produce a fix for each.

---

## Audit Process (Execute In This Order)

### Phase 1: Faithfulness Check (Spec vs Working Doc)
Go through the working doc requirement by requirement:
- Is each requirement represented in the spec?
- Is each requirement represented accurately (not reinterpreted)?
- Has the spec added anything the working doc didn't ask for?

Classify every deviation as:
- **ADDITION**: Spec includes something not in working doc → must be removed
- **OMISSION**: Working doc requires something spec doesn't cover → must be added
- **REINTERPRETATION**: Spec changed the meaning or approach → must be corrected

### Phase 2: Ambiguity & Drift Scan
Flag every instance where an implementing LLM would have to guess:
- **Vague language**: "etc.", "as needed", "properly", "should" where "must" is intended
- **Naming gaps**: Functions described but not named, types not specified
- **Decision gaps**: Implementer must choose between approaches
- **Test gaps**: Functions without test scenarios, happy-path-only tests

### Phase 3: Scorecard
Score every item PASS / PARTIAL / FAIL:

| # | Category | Check | Grade | Justification | Required Fix |
|---|----------|-------|-------|---------------|-------------|
| 1 | Faithfulness | Spec implements exactly what the working doc describes | | | |
| 2 | Contracts | Every payload has canonical JSON shape with types | | | |
| 3 | Contracts | Every DB change has full runnable SQL with comments | | | |
| 4 | Completeness | Every step has concrete "done when" assertion | | | |
| 5 | Completeness | Every file in patch list has specific changes listed | | | |
| 6 | Existing Patterns | At least one pattern reference per major component | | | |
| 7 | Negative Constraints | Section 0 has feature-specific "do NOT" list | | | |
| 8 | Code Quality | All new names explicitly specified | | | |
| 9 | Code Quality | Comment requirements stated per function | | | |
| 10 | Verification | Each step has testable verification | | | |
| 11 | Minimalism | No feature flags, no old/new paths, no unnecessary files | | | |
| 12 | Scope Boundary | Explicit in/out of scope | | | |
| 13 | Current State | Exact existing files/functions identified with paths | | | |
| 14 | Before → After | Behavior changes shown concretely | | | |
| 15 | Testing | Test scenarios with exact inputs and expected outputs | | | |
| 16 | Testing | Tests cover happy path, empty, error, and boundary | | | |

**Grading:**
- 16/16 PASS → IMPLEMENTABLE
- Any PARTIAL or FAIL → NOT IMPLEMENTABLE YET

### Phase 4: Risk Analysis
Top risks if implemented as-is, each with a specific spec edit to mitigate.

### Phase 5: Step-by-Step Review
For each implementation step:
1. Can it be tested independently?
2. Does it depend on anything not built in a previous step?
3. Does it require changes to files NOT in the patch list?
4. Is the verification check runnable and specific?
5. Test coverage sufficient?
6. LLM pitfall prediction: most likely mistake + spec edit to prevent it

---

## Output Format

### SECTION 1: Verdict
One of:
- `IMPLEMENTABLE` — No issues, or info-only. Ship it.
- `IMPLEMENTABLE_WITH_NOTES` — Minor/info issues logged. Proceed unless human overrides.
- `NOT_IMPLEMENTABLE_YET` — Major or critical issues exist. Must fix.
- `ESCALATE` — Auditor unsure, or issues require human judgment.

Include `blocker_count` (count of critical + major issues). If blocker_count = 0, verdict must be IMPLEMENTABLE or IMPLEMENTABLE_WITH_NOTES.

### SECTION 2: Faithfulness Report
Table of OMISSION / ADDITION / REINTERPRETATION findings with exact fixes.

### SECTION 3: Ambiguity & Drift Map
Table: Spec Location | Problem | What LLM will likely do | Rewrite

### SECTION 4: Scorecard
The 16-item table, fully filled in.

### SECTION 5: Risk Register
Table: Risk | Severity | What breaks | Spec edit to mitigate

### SECTION 6: Step-by-Step Review
Per step: Isolation | Dependencies | Hidden coupling | Verification | Test coverage | LLM pitfall

### SECTION 7: Consolidated Fix List
Every fix from all sections, deduplicated and prioritized:

FIX 1 (CRITICAL): [Section X]
Current text: "[quote]"
Replace with: "[exact new text]"
Reason: [why]

FIX 2 (HIGH): [Section X]
...

Priority: CRITICAL → HIGH → MEDIUM

### SECTION 8: Issue Ledger Status (Re-Audits Only)
If this is a re-audit, include a summary table:

| Issue ID | Previous Status | Current Status | Notes |
|----------|----------------|----------------|-------|

This enables convergence tracking across iterations.
```

---

### 9.4 `prompts/prompt3-post-audit.md` — Post-Implementation Audit

```markdown
You are a senior staff engineer performing a post-implementation audit. A feature has been implemented based on a spec. Your job is to do two things:

1. **Verify the implementation matches the spec** (compliance)
2. **Find bugs, edge cases, and issues the spec didn't anticipate** (bug hunting)

Both are equally important.

## Scope Boundary (CRITICAL)

**Audit ONLY against the spec and working doc requirements.** Do not:
- Flag issues in code that wasn't touched by this feature
- Expand scope to adjacent systems not covered by the spec
- Raise "nice to have" improvements as compliance issues
- Treat pre-existing technical debt as a finding

If something outside the spec's scope concerns you, flag it as severity `info` with a note that it's out of scope.

## Re-Audit Protocol (For Iterations 2+)

If you are provided a **previous issue ledger**, you MUST:
1. **Go through each previous issue by ID** and verify if it's resolved in the current code
2. Mark each as `resolved` or `unresolved` in your output
3. **Only flag NEW issues** if they were **introduced by the fix** — do not expand scope
4. New issues get status `new` and a fresh ID continuing the sequence
5. Do NOT re-raise issues that were already resolved
6. Your verdict should reflect the current state: if all blockers are resolved, the code is compliant even if minor/info items remain

## Audit Inputs

You will receive:
1. **The spec** — the contract the implementation must satisfy
2. **The implementation** — Git diff of changes from the base branch to the feature branch

Read the spec first. Read the implementation second. Then audit.

---

## AUDIT PROCEDURE

### Phase 1: Build the Audit Checklist
Extract from the spec every atomic, verifiable requirement. Group by:
- DB changes (tables, columns, types, indexes, constraints, RLS, RPCs)
- New functions/endpoints (name, location, inputs, outputs, behavior)
- Modified functions (what changed, new behavior)
- Data contracts (payload shapes, required fields, types)
- Integration points (what calls what)
- Error handling (what errors, how handled)
- Test requirements (what tests must exist)

### Phase 2: Spec Compliance Audit
For each checklist item:
- **Requirement**: What the spec says
- **Code location**: File path and function/line
- **Status**: MATCH / MISMATCH / MISSING
- **Evidence**: What the code does
- **Fix needed**: If not MATCH

### Phase 3: Code Quality Review
- **Completeness**: Stubs, TODOs, empty catch blocks, placeholders?
- **Comments**: Docstrings on new functions? Intent comments on complex blocks?
- **Pattern consistency**: Matches existing repo patterns?
- **Dead code/bloat**: Unreachable code, unnecessary files, feature flags?

### Phase 3.5: Test Audit
- Test coverage: Does each function/endpoint have tests?
- Test quality: Do tests check spec behavior (not just implementation)?
- Missing tests: For each bug found, is there a test that should catch it?

### Phase 4: Bug Hunting
For each new/modified function:
- **4a: Trace every code path** (happy, branch, early return, error)
- **4b: Check input boundaries** (null, empty, zero, negative, wrong type)
- **4c: Check external dependencies** (error, empty result, partial data, timeout)
- **4d: Check data transformations** (field preservation, null propagation, type coercion)
- **4e: Check state mutations** (idempotency, race conditions, validation, transactions)
- **4f: Check integration seams** (type matching, field name matching, error propagation)

### Phase 5: Verification Audit
For each "done when" assertion: can it be proven with current code?

### Phase 6: Security Spot Check
- SQL injection vectors?
- Missing auth checks?
- Sensitive data in logs/responses?
- RLS configured for new tables?
- Hardcoded secrets?
- Input validation?

---

## OUTPUT FORMAT

### SECTION 1: Verdict
One of:
- `COMPLIANT` — No issues, or info-only. Ship it.
- `COMPLIANT_WITH_NOTES` — Minor/info issues logged. Proceed unless human overrides.
- `NOT_COMPLIANT` — Major or critical issues. Must fix.
- `ESCALATE` — Auditor unsure, or issues require human judgment.

Include `blocker_count` (count of critical + major issues/bugs). If blocker_count = 0, verdict must be COMPLIANT or COMPLIANT_WITH_NOTES.

One paragraph summary.

### SECTION 2: Compliance Matrix
| # | Spec Requirement | Code Location | Status | Evidence | Fix Needed |

### SECTION 3: Bugs Found
For each bug:
BUG [N]: [Title]
Severity: CRITICAL / HIGH / MEDIUM / LOW
Location: [file:function:line]
Description: [What's wrong]
Trigger: [What input/state causes it]
Impact: [What happens]
Test exists: YES/NO
Fix: [Exact code change]
Test to add: [Input, expected output, assertion]

### SECTION 4: Code Quality Issues
QUALITY [N]: [Title]
Location: [file:function]
Issue: [What's wrong]
Fix: [Specific change]

### SECTION 5: Test Audit Results
| # | Function/Endpoint | Test Exists | Location | Quality | Issues |

### SECTION 6: Verification Status
| # | Verification Step | Provable? | How to Prove | Missing |

### SECTION 7: Consolidated Fix List
Every fix, deduplicated, ordered by priority:
FIX 1 (CRITICAL): [Title]
File: [path]
Function: [name]
Current: [what code does]
Change: [exact change]
Reason: [reference to finding]
Test: [test to add/fix]

Priority: CRITICAL → HIGH → MEDIUM → LOW

### SECTION 8: Issue Ledger Status (Re-Audits Only)
If this is a re-audit, include a summary table:

| Issue ID | Previous Status | Current Status | Notes |
|----------|----------------|----------------|-------|

This enables convergence tracking across iterations.
```

---

### 9.5 `prompts/implementation-primer.md` — Implementation Instructions

This gets prepended to the spec when handing it to the coding agent for implementation. **You MUST customize the Environment Constraints section for YOUR project.**

```markdown
You are implementing a feature based on a detailed spec. Follow these rules:

1. **Follow the spec exactly.** Do not add features, optimizations, or abstractions not in the spec.
2. **Complete implementations only.** No stubs, no TODOs, no placeholders.
3. **Match existing patterns.** Read project conventions and existing code before writing anything new.
4. **Handle all errors.** Every external call needs error handling as specified.
5. **Write tests alongside code.** Each function gets tests as specified in the spec.
6. **Comment non-trivial code.** Docstrings for functions, inline comments for complex logic.
7. **Verify your work.** Run build and lint before finishing.
8. **Full spec output on revisions.** If asked to revise the spec based on feedback, you MUST return the complete, fully updated spec end-to-end — not a patch, not a diff, not a summary of changes.

## Environment Constraints

⚠️ CUSTOMIZE THESE FOR YOUR PROJECT. These are examples from the original deployment:

- **Port binding may be blocked (EPERM).** Do not attempt `next dev`, `next start`, or any server that binds ports.
- **Test route handlers via direct `node` invocation** with mock `Request` objects. Do not use curl/fetch against a running server.
- **Do not modify `.claude/settings.local.json`.** If changed temporarily, revert before finishing.
- **Build verification:** `npm run build` works fine. Use it freely.
- **Lint:** `npm run lint` may fail if no ESLint config exists in the repo. This is a pre-existing condition, not your fault. Note it and move on.

Replace or extend these with your own project's constraints. Common things to document:
- Commands that don't work in your environment
- Files that must not be modified
- Services that aren't available (databases, APIs, etc.)
- Sandbox restrictions

## Codebase Integrity Safeguards

- **Missing expected code is a mismatch, not a scaffold task.** If the spec references files, modules, functions, or patterns that don't exist in the codebase, that's a spec/codebase mismatch — NOT an invitation to invent architecture to fill the gaps. Stop and report the discrepancy.
- **Stop if the spec describes existing files you can't find.** If the spec says "modify `src/lib/foo.ts`" and that file doesn't exist, do NOT create it from scratch or guess what it should contain. Report the mismatch immediately.
- **Never scaffold missing infrastructure.** If the spec assumes a database table, API route, auth system, or service that isn't in the codebase, flag it — don't build it unless it's explicitly part of the spec's deliverables.
- **Verify imports and dependencies exist** before writing code that depends on them. If an import target doesn't exist, stop and report it.

When you encounter any of these mismatches, output a clear list of discrepancies as your final message instead of proceeding with implementation.

## Git / Branch Rules

**CRITICAL:** All feature work MUST branch off the configured base branch (e.g., `staging`) — NEVER off `main` unless main IS the base branch.

Before writing any code:
1. Run `git fetch origin`
2. Confirm you are on the correct feature branch
3. **Never commit directly to the base branch or `main`**

Read the codebase. Understand the existing structure. Then implement.
```

---

## 10. Schema Files (Full Content)

### 10.1 `schemas/spec-audit-verdict.json`

```json
{
  "type": "object",
  "properties": {
    "verdict": {
      "type": "string",
      "enum": ["IMPLEMENTABLE", "IMPLEMENTABLE_WITH_NOTES", "NOT_IMPLEMENTABLE_YET", "ESCALATE"],
      "description": "IMPLEMENTABLE: no issues or info-only. IMPLEMENTABLE_WITH_NOTES: minor/info issues logged. NOT_IMPLEMENTABLE_YET: major or critical issues. ESCALATE: auditor unsure or needs human judgment."
    },
    "summary": {
      "type": "string",
      "description": "One-paragraph summary of spec quality"
    },
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": { "type": "string", "description": "Stable issue ID, e.g. SPEC-1. Persists across iterations." },
          "section": { "type": "string" },
          "severity": { "type": "string", "enum": ["critical", "major", "minor", "info"] },
          "issue": { "type": "string" },
          "suggested_fix": { "type": "string" },
          "status": { "type": "string", "enum": ["new", "unresolved", "resolved"], "description": "For re-audits: resolved/unresolved/new." }
        },
        "required": ["id", "section", "severity", "issue", "suggested_fix", "status"],
        "additionalProperties": false
      }
    },
    "blocker_count": {
      "type": "integer",
      "description": "Count of critical + major issues. If 0, spec is implementable."
    }
  },
  "required": ["verdict", "summary", "issues", "blocker_count"],
  "additionalProperties": false
}
```

### 10.2 `schemas/impl-audit-verdict.json`

```json
{
  "type": "object",
  "properties": {
    "verdict": {
      "type": "string",
      "enum": ["COMPLIANT", "COMPLIANT_WITH_NOTES", "NOT_COMPLIANT", "ESCALATE"],
      "description": "COMPLIANT: no issues or info-only. COMPLIANT_WITH_NOTES: minor/info logged. NOT_COMPLIANT: major or critical. ESCALATE: needs human judgment."
    },
    "summary": { "type": "string" },
    "compliance_issues": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": { "type": "string", "description": "Stable ID, e.g. IMPL-C1. Persists across iterations." },
          "spec_section": { "type": "string" },
          "severity": { "type": "string", "enum": ["critical", "major", "minor", "info"] },
          "issue": { "type": "string" },
          "file": { "type": "string" },
          "fix": { "type": "string" },
          "status": { "type": "string", "enum": ["new", "unresolved", "resolved"] }
        },
        "required": ["id", "spec_section", "severity", "issue", "file", "fix", "status"],
        "additionalProperties": false
      }
    },
    "bugs": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": { "type": "string", "description": "Stable ID, e.g. IMPL-B1. Persists across iterations." },
          "severity": { "type": "string", "enum": ["critical", "major", "minor", "info"] },
          "file": { "type": "string" },
          "line_range": { "type": "string" },
          "description": { "type": "string" },
          "fix": { "type": "string" },
          "status": { "type": "string", "enum": ["new", "unresolved", "resolved"] }
        },
        "required": ["id", "severity", "file", "line_range", "description", "fix", "status"],
        "additionalProperties": false
      }
    },
    "blocker_count": {
      "type": "integer",
      "description": "Count of critical + major issues/bugs. If 0, implementation is compliant."
    }
  },
  "required": ["verdict", "summary", "compliance_issues", "bugs", "blocker_count"],
  "additionalProperties": false
}
```

---

## 11. Verdict System & Iteration Logic

### Severity Tiers

| Severity | Blocks Pipeline? | Description |
|----------|-----------------|-------------|
| `critical` | Yes | Security hole, data loss, broken core flow |
| `major` | Yes | Significant gap, must fix |
| `minor` | No | Improvement, edge case, style |
| `info` | No | Observation, suggestion, out of scope |

### Spec Audit Verdicts

| Verdict | Meaning | Action |
|---------|---------|--------|
| `IMPLEMENTABLE` | No issues, or info-only | → Phase 4 |
| `IMPLEMENTABLE_WITH_NOTES` | Minor/info logged | → Phase 4 |
| `NOT_IMPLEMENTABLE_YET` | Major/critical exist | Fix → re-audit |
| `ESCALATE` | Auditor unsure | Pause, notify human |

### Impl Audit Verdicts

| Verdict | Meaning | Action |
|---------|---------|--------|
| `COMPLIANT` | No issues, or info-only | → Phase 6 |
| `COMPLIANT_WITH_NOTES` | Minor/info logged | → Phase 6 |
| `NOT_COMPLIANT` | Major/critical exist | Fix → re-audit |
| `ESCALATE` | Auditor unsure | Pause, notify human |

### Issue Ledgers Drive Convergence

**This is the single most important design decision in the pipeline.** Without issue ledgers:
- Auditors find new tangential issues each iteration instead of verifying fixes
- The audit loop never converges (death spiral)
- The orchestrator can't tell if progress is being made

With issue ledgers:
1. After each audit, extract issues into a ledger file (plain text, one per line)
2. Feed the ledger to the revision/fix step (so it knows EXACTLY what to fix — nothing more)
3. Feed the ledger to the re-audit step (so it verifies those specific issues — not new tangents)
4. Re-audit marks each issue as `resolved`, `unresolved`, or `new`
5. Only `new` issues are ones introduced by the revision — no scope creep

### Max Iteration Auto-Promotion

When the iteration cap is reached:
- **blocker_count = 0** → Auto-promote to `WITH_NOTES` and proceed. Log residual minor/info issues.
- **blocker_count > 0** → `ESCALATE` to human. Pipeline pauses. Do NOT proceed.

---

## 12. JSONL Parsing Reference (Codex CLI Output Format)

When you run `codex exec --json`, it outputs newline-delimited JSON (JSONL). Each line is a separate JSON object representing an event.

### Key Event Types

```jsonc
// First line — contains thread_id (CRITICAL: save this for session resume)
{"type":"thread.started","thread_id":"<uuid>"}

// Turn lifecycle
{"type":"turn.started"}
{"type":"turn.completed","usage":{"input_tokens":...,"output_tokens":...}}
{"type":"turn.failed","error":{"message":"..."}}

// Agent output (text messages from the coding agent)
{"type":"item.completed","item":{"type":"agent_message","text":"..."}}

// Shell command execution
{"type":"item.completed","item":{"type":"command_execution","command":"...","exit_code":0}}

// Fatal error
{"type":"error","message":"..."}
```

### Critical Extractions

**Get thread_id (for session resume):**
```bash
THREAD_ID=$(head -1 "$JSONL_LOG" | jq -r '.thread_id')
```

**Get last agent message (when `-o` isn't available):**
```bash
grep '"type":"agent_message"' "$JSONL_LOG" | tail -1 | jq -r '.item.text'
```

**Check for errors:**
```bash
grep '"turn.failed"' "$JSONL_LOG" | jq -r '.error.message'
```

### `exec resume` Limitations

`codex exec resume <thread_id>` does NOT support these flags:
- `-C <dir>` — the session remembers the original directory
- `-o <file>` — must instruct the agent to write files via shell commands
- `--output-schema <file>` — must include the schema inline in the prompt text

This is why re-audit prompts include the schema JSON directly and tell the agent to write the verdict file via `cat > file.json << 'EOF'`.

---

## 13. Error Handling & Escalation

### Recoverable Errors

| Error | Action |
|-------|--------|
| Codex timeout / non-zero exit | Retry once with same prompt |
| JSONL parse failure | Skip malformed line, retry once if critical data missing |
| `turn.failed` event | Extract error message, retry once |
| `thread_id` not found | Start new session, note discontinuity in state.json |
| Build/lint failure after fix | Re-resume impl session (counts toward iterations) |

### Escalation Triggers → `status: "needs_human"`
- Same audit verdict 2x in a row with identical unresolved issues (loop not converging)
- Max iterations reached with `blocker_count > 0`
- Git operation failures (merge conflict, push rejection)
- Codex model-level errors (auth failure, rate limit, model unavailable)
- Codebase integrity mismatch (spec references things that don't exist)
- Unrecognized errors you can't categorize

### Escalation Message Template
Send this via the message tool immediately:
```
⚠️ Pipeline needs attention: <FEATURE_NAME>

Phase: <CURRENT_PHASE>
Issue: <ERROR_DESCRIPTION>
Branch: <BRANCH>
State file: state/<slug>/state.json

Pipeline is paused. Options:
1. Fix manually and tell me to resume from phase <NEXT_PHASE>
2. Tell me to retry current phase
3. Tell me to abort
```

---

## 14. Notification Wiring (OpenClaw message tool)

All pipeline notifications use OpenClaw's `message` tool. Here's the exact pattern:

### Sending a Notification
Use the `message` tool with these parameters:
```
message(
  action="send",
  channel="<your_notification_channel>",   # e.g., "webchat", "discord", "telegram"
  message="<notification text>"
)
```

### When to Notify
| Event | Notification |
|-------|-------------|
| Pipeline starts (Phase 1) | "🚀 Starting pipeline for '<feature>' — created branch feature/<slug>" |
| Spec generated (Phase 2) | "📝 Spec generated for '<feature>' — starting audit" |
| Spec audit passed (Phase 3) | "✅ Spec passed audit (iteration N/max) — starting implementation" |
| Implementation done (Phase 4) | "🔨 Implementation complete for '<feature>' — running build verification" |
| Build passes (Phase 4.5) | "🏗️ Build/lint passed — starting post-implementation audit" |
| Impl audit passed (Phase 5) | "✅ Code passed audit (iteration N/max) — finalizing" |
| Pipeline complete (Phase 6) | Full summary (branch, iterations, files changed, next steps) |
| Escalation | ⚠️ escalation template (see Section 13) |

### Critical Rules
- **Send completion notifications immediately** — don't fold them into heartbeat responses
- **Use the channel name** (e.g., `"webchat"`, `"discord"`) — NOT session labels or conversation IDs
- **If your reply IS the notification**, respond with `NO_REPLY` to avoid duplicate messages

---

## 15. AGENTS.md & HEARTBEAT.md Integration

### 15.1 What to Add to AGENTS.md

Add a section like this to your `AGENTS.md` so you know to run the pipeline when triggered:

```markdown
### Pipeline Sessions
If the human asks you to run a pipeline, sends a working doc, or you detect a new file in `pipeline/inbox/`:
- Read `pipeline/PIPELINE-SPEC.md` — this is your operational runbook
- Check `pipeline/state/` for any in-progress features
- Follow the runbook exactly. Do not improvise.
- Proceed through phases automatically — don't wait for human permission between phases
- Only escalate on failures or "needs_human" verdicts
```

### 15.2 How to Trigger the Pipeline

Your human can trigger the pipeline in several ways:

1. **Direct message:** "Run the implementation pipeline on this working doc: [path or pasted content]"
2. **Drop a file in inbox:** Place a `.md` file in `pipeline/inbox/` — your heartbeat can detect it
3. **Explicit command:** "Start pipeline for <feature-name> using <path-to-working-doc>"

When triggered:
1. Read `pipeline/PIPELINE-SPEC.md` (the runbook)
2. Execute Phase 1 through Phase 6 in order
3. Do NOT wait for human approval between phases (unless escalation is needed)
4. Send notifications at each milestone

### 15.3 Inbox Auto-Detection (Optional)

Add this to your `HEARTBEAT.md` to auto-detect new working docs:

```markdown
## Pipeline Inbox Check
Check `pipeline/inbox/` for any `.md` files.
If found:
- Move the file to `pipeline/state/<slug>/working-doc.md`
- Start the pipeline
- Delete or rename the inbox file to prevent re-processing (e.g., append `.processed`)
```

---

## 16. Heartbeat Recovery Integration

Add this to your `HEARTBEAT.md`. This is the **primary recovery mechanism** — if OpenClaw's context resets mid-pipeline (compaction, sleep, restart), the next heartbeat detects the stalled state and resumes.

```markdown
## Priority 1: Pipeline Status Check
Check `pipeline/state/` for any feature directories.
For each one, read `state.json`:
- If `status: "in_progress"` → check `resume_hint` and thread ID fields first.
  - If a coding agent session exists → check if still running (poll the exec process). If still running, do nothing. If finished, process results per the current `phase`.
  - If no active session → the pipeline was interrupted. Read `pipeline/PIPELINE-SPEC.md`, then resume from the current `phase`.
  - **When a session finishes:** ALWAYS re-read PIPELINE-SPEC.md before taking any action. Follow the phase sequence exactly. Do NOT skip phases. Do NOT commit/push until the runbook says to.
- If `status: "needs_human"` → remind the human if not already notified.
- If `status: "completed"` or `status: "failed"` → no action needed.

If resuming a pipeline, DO NOT reply HEARTBEAT_OK — execute the resume and report status.
```

### Why This Works
- `state.json` persists across sessions — it's on disk, not in memory
- `resume_hint` tells the next session exactly what was happening and what to do next
- Thread IDs allow resuming coding agent sessions that are still alive
- The heartbeat fires periodically (every 5-30 minutes), so recovery happens automatically

---

## 17. Concurrency (Multiple Features)

Multiple features can run through the pipeline concurrently. Each has its own:
- **Git branch** (`feature/<slug-a>`, `feature/<slug-b>`)
- **State directory** (`state/<slug-a>/`, `state/<slug-b>/`)
- **Thread IDs** (separate coding agent sessions)
- **Iteration counters** (independent tracking)

### How to Handle Concurrent Runs
- While waiting on a long-running coding agent process for Feature A, you can advance Feature B's next phase
- Each feature's state is completely independent — no shared state
- Git branch switching: always `git checkout` to the right branch before running commands for a specific feature
- Watch out for: coding agent sessions that modify `.git` state — don't let two sessions work in the same repo simultaneously unless they're on different branches

### Practical Limits
- Coding agent API rate limits may bottleneck concurrent runs
- Git conflicts are unlikely (separate branches) but check on merge
- Monitor token usage — each feature run can consume significant tokens across all phases

---

## 18. Operational Runbook Template (PIPELINE-SPEC.md)

**This is the document you (the orchestrator AI) read at the start of every pipeline run.** Copy this template, replace all `$VARIABLES` with your actual values from `config.yaml`, and save it as `pipeline/PIPELINE-SPEC.md`.

The runbook is your standing orders. Follow it mechanically. Do not improvise.

```markdown
# Implementation Pipeline — Operational Runbook

**Purpose:** Standing orders for the pipeline orchestrator. Read this at the start of every pipeline run. If context has reset, this document tells you everything you need to know to execute.

---

## Overview

You are the **orchestrator**. You do NOT write code. You run the coding agent CLI against the repo, and it writes the code. You manage the pipeline phases, parse output, track state, and escalate when things go wrong.

```
Working Doc → Spec Gen → Spec Audit Loop → Implementation → Build Verify → Post-Audit Loop → Finalize
```

---

## Paths & Config

Replace these with your actual values:

```
REPO_PATH=$REPO_PATH
CODEX_BIN=$CODEX_BIN
MODEL=$MODEL
PIPELINE_DIR=$PIPELINE_DIR
PROMPTS_DIR=$PROMPTS_DIR
SCHEMAS_DIR=$SCHEMAS_DIR
STATE_DIR=$STATE_DIR
LOGS_DIR=$LOGS_DIR
INBOX_DIR=$INBOX_DIR
BASE_BRANCH=$BASE_BRANCH
BRANCH_PREFIX=$BRANCH_PREFIX
BUILD_COMMAND=$BUILD_COMMAND
LINT_COMMAND=$LINT_COMMAND
NOTIFICATION_CHANNEL=$NOTIFICATION_CHANNEL

MAX_SPEC_ITERATIONS=$MAX_SPEC_ITERATIONS
MAX_IMPL_ITERATIONS=$MAX_IMPL_ITERATIONS
```

---

## Execution Rules

1. **I am a project manager, not a coder.** I orchestrate the coding agent, I don't write code myself.
2. **Follow the pipeline exactly.** No skipping phases, no exceeding iteration limits without escalating.
3. **Preserve all artifacts.** Every JSONL log, every audit result, every state file.
4. **When in doubt, escalate.** A paused pipeline beats a broken branch.
5. **Never push to the base branch or `main`.** Feature branches only. Human merges.
6. **Session continuity.** Auditors stay in the same thread across iterations. Use `exec resume`, not fresh `exec`.
7. **Issue ledgers drive convergence.** Extract issues after every audit. Feed them back to revisions and re-audits.
8. **Full spec output on every revision.** Never accept patches or diffs. Demand the complete updated spec.
9. **Severity determines blocking.** Only critical/major issues block. Minor/info are logged, not blocking.
10. **Update resume_hint after every state change.** Future-you depends on it.
11. **Send notifications at milestones.** Use the message tool with channel="$NOTIFICATION_CHANNEL". Don't bury updates in heartbeat responses.

---

## Phase Execution

[Copy the exact phase commands from Section 7 of this document, with $VARIABLES replaced by your actual paths]

---

## Session Startup Checklist

When context resets or a new session starts:
1. Read this file (PIPELINE-SPEC.md)
2. Check `state/` for any in-progress features
3. For each in-progress feature, read its `state.json` to determine where it left off
4. Read `resume_hint` for orientation
5. Resume from the appropriate phase
6. If unclear, ask the human
```

---

## 19. Example Working Doc

Here's a realistic example of what a working doc looks like — the input that triggers the pipeline. This helps you understand the expected format and level of detail.

```markdown
# Add User Notification Preferences

## Goal
Allow users to configure which notifications they receive and through which channels (email, in-app, push). Currently all users get all notifications with no way to opt out or customize.

## Context
- Users have complained about notification spam (GitHub issue #142)
- We currently send notifications via a single `sendNotification()` function in `src/lib/notifications.ts`
- No user preferences exist — every event triggers every notification type
- We use Supabase for auth and database

## Current State
- `src/lib/notifications.ts` — single `sendNotification(userId, event, message)` function
- `src/app/api/notifications/route.ts` — POST endpoint that calls sendNotification
- No `notification_preferences` table exists
- Users table: `public.users` with columns: id, email, name, created_at

## How It Should Work
1. New DB table `notification_preferences` with columns: user_id, channel (email/push/in_app), event_type, enabled (boolean)
2. Default: all notifications enabled for new users (insert defaults on first login)
3. API endpoint: GET/PUT `/api/notifications/preferences` — read and update preferences
4. UI: Settings page with toggle matrix (event types × channels)
5. `sendNotification` checks preferences before sending — skips disabled channels
6. If no preference row exists for a user+event+channel combo, treat as enabled (backward compatible)

## Technical Approach
- Add Supabase migration for the new table with RLS (users can only read/write their own prefs)
- Modify `sendNotification` to query preferences first
- New React component `NotificationSettings` using the existing settings page layout
- Follow the pattern of the existing `ProfileSettings` component for layout and data fetching

## What NOT to Change
- Don't touch the notification sending infrastructure (email provider, push service)
- Don't change the events themselves — only whether they're delivered
- Don't add notification grouping or scheduling — that's a separate feature

## Scope
- IN: preferences table, API, UI settings page, sendNotification filter
- OUT: notification templates, delivery infrastructure, notification history/log

## Edge Cases
- User deletes account → cascade delete preferences (FK constraint)
- Concurrent preference updates → last-write-wins is fine
- Missing preference row → treat as enabled
```

This working doc:
- Explains WHAT and WHY clearly
- References existing code by file path
- Makes design decisions (not "we could do X or Y" but "do X")
- Defines scope explicitly
- Lists what NOT to change
- Covers edge cases

---

## 20. Step-by-Step Execution Checklist (Per Feature)

When triggered with a working doc, follow this checklist in order. Check each box as you complete it.

### Pre-Flight
- [ ] Read `pipeline/PIPELINE-SPEC.md` (your operational runbook)
- [ ] Read the working doc
- [ ] Derive the feature slug
- [ ] Verify the repo is accessible: `cd $REPO_PATH && git status`
- [ ] Verify the coding agent is accessible: `$CODEX_BIN --version` or equivalent

### Phase 1: Setup
- [ ] `git fetch origin`
- [ ] `git checkout $BASE_BRANCH && git pull origin $BASE_BRANCH`
- [ ] `git checkout -b feature/$SLUG $BASE_BRANCH`
- [ ] `mkdir -p $STATE_DIR/$SLUG`
- [ ] Copy working doc to `$STATE_DIR/$SLUG/working-doc.md`
- [ ] Create `state.json` with initial values (phase: "setup", status: "in_progress")
- [ ] Update `resume_hint`: "Phase 1 complete. Ready for spec generation."
- [ ] Send notification: pipeline started

### Phase 2: Spec Generation
- [ ] Assemble prompt: `prompt1-spec-writer.md` + working doc content
- [ ] Run `codex exec` with assembled prompt, `--json`, `--full-auto`, `-o spec.md`
- [ ] Verify `spec.md` was written (check file exists and is non-empty)
- [ ] Extract `thread_id` from JSONL log first line
- [ ] Update state.json: `spec_thread_id`, `phase: "spec_gen"`
- [ ] Update `resume_hint`: "Spec generated. Ready for spec audit."
- [ ] Send notification: spec generated

### Phase 3: Spec Audit Loop
For each iteration (1 to max_spec_iterations):
- [ ] If iteration 1: run fresh `codex exec` with `prompt2-spec-auditor.md` + working doc + spec + `--output-schema`
- [ ] If iteration 2+: run `codex exec resume $SPEC_AUDIT_THREAD_ID` with previous issue ledger + schema inline
- [ ] Extract/save audit thread ID (iteration 1 only)
- [ ] Parse verdict JSON: extract `verdict` and `blocker_count`
- [ ] Extract issue ledger to `spec-issues-$N.md`
- [ ] **If IMPLEMENTABLE or IMPLEMENTABLE_WITH_NOTES:** break loop → proceed to Phase 4
- [ ] **If NOT_IMPLEMENTABLE_YET:** run spec revision via `codex exec resume $SPEC_THREAD_ID`, then continue loop
- [ ] **If ESCALATE:** set status "needs_human", send escalation notification, STOP
- [ ] **If max iterations reached:** check blocker_count. If 0 → auto-promote. If >0 → ESCALATE, STOP.
- [ ] Update state.json: `spec_iteration`, `spec_verdict`, `phase: "spec_audit"`
- [ ] Update `resume_hint` after each iteration
- [ ] Send notification: spec audit passed
- [ ] Generate context brief

### Phase 4: Implementation
- [ ] Assemble prompt: `implementation-primer.md` + spec content
- [ ] Run `codex exec` with assembled prompt, `--json`, `--full-auto`
- [ ] Check output — if mismatch report instead of implementation → escalate
- [ ] Extract `thread_id` from JSONL log
- [ ] Update state.json: `impl_thread_id`, `phase: "implementation"`
- [ ] Update `resume_hint`
- [ ] Send notification: implementation complete

### Phase 4.5: Build Verification
- [ ] Run `$BUILD_COMMAND`, capture output to `build-log.txt`
- [ ] Run `$LINT_COMMAND`, capture output to `lint-log.txt`
- [ ] **If both pass:** update state.json: `build_passed: true`, `lint_passed: true`, `phase: "build_verify"`
- [ ] **If either fails:** resume impl session with error output, re-run build/lint, increment `impl_iteration`
- [ ] Update `resume_hint`
- [ ] Send notification: build verification passed

### Phase 5: Post-Impl Audit Loop
For each iteration (1 to max_impl_iterations):
- [ ] Get diff: `git diff $BASE_BRANCH..HEAD`
- [ ] If iteration 1: run fresh `codex exec` with `prompt3-post-audit.md` + spec + diff + `--output-schema`
- [ ] If iteration 2+: run `codex exec resume $IMPL_AUDIT_THREAD_ID` with previous issue ledger + new diff + schema inline
- [ ] Extract/save audit thread ID (iteration 1 only)
- [ ] Parse verdict JSON: extract `verdict` and `blocker_count`
- [ ] Extract issue ledger to `impl-issues-$N.md`
- [ ] **If COMPLIANT or COMPLIANT_WITH_NOTES:** break loop → proceed to Phase 6
- [ ] **If NOT_COMPLIANT:** run code fix via `codex exec resume $IMPL_THREAD_ID`, then re-run build verification, continue loop
- [ ] **If ESCALATE:** set status "needs_human", send escalation notification, STOP
- [ ] **If max iterations reached:** check blocker_count. If 0 → auto-promote. If >0 → ESCALATE, STOP.
- [ ] Update state.json: `impl_iteration`, `impl_verdict`, `phase: "impl_audit"`
- [ ] Update `resume_hint` after each iteration
- [ ] Send notification: impl audit passed
- [ ] Generate context brief

### Phase 6: Finalize
- [ ] `git add -A` (with exclusions for .env, logs, node_modules, .DS_Store)
- [ ] `git commit` with descriptive message including iteration counts
- [ ] `git push origin feature/$SLUG`
- [ ] Generate `summary.md` with full report
- [ ] Send notification: full summary with branch name, files changed, next steps
- [ ] Update state.json: `phase: "completed"`, `status: "completed"`
- [ ] Update `resume_hint`: "Pipeline complete. Branch pushed. Awaiting human review."

### Post-Run
- [ ] Verify branch exists on remote: `git ls-remote origin feature/$SLUG`
- [ ] Log the run in your memory files for future reference

---

## 21. Lessons Learned & Gotchas

These are from real production runs. Save yourself the debugging time.

1. **`exec resume` doesn't support `-o` or `--output-schema`.** Include the schema inline in the prompt and instruct the agent to write the file via shell command. This is the #1 thing that will bite you.

2. **Spec audit death spiral.** Without issue ledgers and scope boundaries, auditors find new tangential issues each iteration instead of verifying fixes. The ledger system (Section 11) fixes this. This was our biggest lesson learned.

3. **Never skip phases.** When a coding session completes, ALWAYS re-read the runbook before acting. Don't improvise phase transitions. The runbook exists to prevent drift.

4. **Full spec output on revisions.** If you tell the spec writer to "fix issues," it might return a patch or diff. Explicitly demand "return the COMPLETE UPDATED SPEC end-to-end — not a patch, not a diff." Every time.

5. **Spawning fresh sessions for re-audits loses context.** Always resume the same thread for iterative work. Fresh sessions don't know what was already discussed.

6. **Pipeline notifications vs heartbeat responses.** Don't bury completion reports in heartbeat responses. The human needs to see them — proactively message on their active channel.

7. **Environment constraints must be codified.** If port binding is blocked, certain commands fail, or services are unavailable in your environment, put that in the implementation primer. Otherwise the coding agent discovers these the hard way, wasting an iteration.

8. **JSON trailing commas.** Coding agents sometimes produce JSON with trailing commas that break `jq`. Watch for this when parsing verdict files.

9. **Codebase integrity safeguards are critical.** Without them, the coding agent will happily scaffold missing infrastructure instead of reporting that the spec doesn't match the codebase. This wastes an entire implementation cycle.

10. **Context briefs save tokens and time.** Generating a brief between phases means resumed sessions don't need to re-read all artifacts from scratch.

11. **Long-running exec processes.** Coding agent sessions (especially implementation) can take 10-30+ minutes. Use background execution and poll for completion. Don't block your main session.

12. **State is king.** If it's not in `state.json`, it didn't happen. Update state after every meaningful action. Your future self (after a context reset) depends entirely on what's on disk.

13. **First run will have issues.** Expect to discover environment-specific constraints on your first pipeline run. That's fine — update `implementation-primer.md` and your runbook after each run.

---

## 22. Adapting to Your Project

This pipeline was built for a Next.js/TypeScript/Supabase project, but the architecture is **project-agnostic**. Here's what to change:

### Build Toolchain
| Your Stack | Build Command | Lint Command | Test Command |
|-----------|--------------|-------------|-------------|
| Node/npm | `npm run build` | `npm run lint` | `npm test` |
| Python | `python -m py_compile` or `mypy .` | `ruff check .` or `flake8` | `pytest` |
| Rust | `cargo build` | `cargo clippy` | `cargo test` |
| Go | `go build ./...` | `golangci-lint run` | `go test ./...` |
| Java/Gradle | `./gradlew build` | `./gradlew check` | `./gradlew test` |

### Branch Strategy
- Change `staging` to whatever your base branch is (`develop`, `main`, etc.)
- Change `feature/` prefix if you use a different convention

### Coding Agent
Swap Codex CLI commands for your agent's CLI. Key requirements:
1. Autonomous mode (no human-in-the-loop during execution)
2. Repo directory access (read + write)
3. Structured output (JSON) or parseable text output
4. Session resume (continue a previous conversation)

If your agent doesn't support session resume, you lose the thread continuity for iterative audits. Workaround: include full context in each prompt instead of relying on conversation memory.

### Spec Structure
The 10-section spec template works for most projects. Customize:
- Remove "Database Migrations" (Section 5) if no DB
- Add "API Schema" section if you're building REST/GraphQL APIs
- Add "Infrastructure" section if you're doing DevOps/IaC work
- Adjust the 16-item scorecard to match your priorities

### Environment Constraints
The most project-specific piece. Update `implementation-primer.md` with:
- Commands that don't work in your sandbox
- Files that must not be modified
- Services that aren't available
- Any restrictions on the coding agent's environment

### The Core That Stays the Same
Regardless of your project, these elements are universal:
- **Phase-gate architecture** (spec → audit → implement → audit → ship)
- **Issue ledger convergence system** (the fix for the death spiral)
- **Structured verdict schemas** (machine-parseable pass/fail with severity)
- **State management** (disk-based, survives context resets)
- **Heartbeat recovery** (automatic resume after interruption)
- **Separation of orchestrator and coder** (PM doesn't write code)

---

*Generated 2026-02-19. Questions → ask the human who gave you this doc.*
