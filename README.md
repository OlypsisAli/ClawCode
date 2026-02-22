# ClawCode

### Give your clawbot a working doc. Get back a feature branch.

ClawCode is an autonomous code implementation framework built for AI agents. You write a plain-English feature description, hand it to your clawbot, and it handles everything — spec generation, spec auditing, implementation, code auditing, bug fixing, and shipping a clean feature branch. You review and merge.

No SDK. No install. No dependencies. It's a single document your clawbot reads and executes.

```
You write a Working Doc (plain English)
        |
  [1] Setup ──────────── branch created, state initialized
        |
  [2] Spec Gen ───────── clawbot reads codebase + working doc, produces implementation spec
        |
  [3] Spec Audit Loop ── separate session audits the spec (up to N rounds, fixes between each)
        |
  [4] Implementation ─── clawbot implements the spec (writes actual code)
        |
  [4.5] Build Verify ─── run build + lint, fix if broken
        |
  [5] Code Audit Loop ── separate session audits the code (up to N rounds, fixes between each)
        |
  [6] Finalize ───────── git commit, push, summary report
        |
  You review the feature branch and merge
```

---

## Quickstart

```
1. Clone this repo
2. Open docs/framework.md
3. Give it to your clawbot
4. Say: "Read this. Set up the pipeline for my project."
5. Write a working doc describing what you want built
6. Say: "Run the pipeline on this working doc: [path]"
7. Go do something else
8. Come back to a feature branch ready for review
```

That's it. The framework doc is self-contained — your clawbot reads it, sets up the directory structure, customizes the config, and runs the pipeline. The prompt files, schemas, and templates in this repo are pre-extracted for convenience, but they're all inside the framework doc too.

---

## Why I built this

I was shipping features on a complex codebase using AI agents and kept hitting the same walls:

**Spec drift** — you ask for one thing, the AI builds three things and "improves" your architecture along the way.

**The 80% problem** — implementation looks done until you notice the stubs, the TODOs, the empty error handlers, the missing edge cases.

**Audit death spirals** — you ask the AI to review its own work. It finds issues. You fix them. It finds *different* issues. You fix those. It finds *new* issues. The loop never converges. You've been going in circles for an hour.

**Context amnesia** — the AI forgets what it was doing mid-implementation. Starts over. Makes different decisions the second time. Nothing is consistent.

**Zero paper trail** — nothing is tracked. No artifacts are saved. When something goes wrong you're debugging blind.

ClawCode fixes all of these with three core ideas:

### 1. Separation of orchestrator and coder
Your clawbot is the **project manager**, not the coder. It manages phases, tracks state, parses verdicts, and escalates. A separate coding agent session does the actual code reading and writing. This prevents the orchestrator from hallucinating code and gives the coding agent full sandbox access.

### 2. Issue ledger convergence
This is the big one. After every audit, issues are extracted into a ledger with stable IDs. The ledger gets fed to both the fix step (fix *exactly these issues*, nothing more) and the re-audit step (verify *these specific issues*, no expanding scope). Re-audits mark each issue as `resolved`, `unresolved`, or `new` — and `new` only counts if the fix *introduced* it. No scope creep. No tangents. The loop converges.

### 3. Disk-based state that survives everything
`state.json` tracks phase, thread IDs, iteration counts, verdicts, and a `resume_hint` — a one-sentence orientation for the next session. If your clawbot's context resets mid-pipeline (compaction, sleep, crash), the next session reads the state file and picks up exactly where it left off. Heartbeat recovery handles this automatically.

---

## What's in this repo

```
clawcode/
├── README.md                          # You're here
├── docs/
│   └── framework.md                   # THE FILE — hand this to your clawbot
│                                      #   Complete, self-contained, 2000+ lines
│                                      #   Everything below is extracted from this
├── prompts/
│   ├── prompt0-working-doc.md         # Interactive working doc creation helper
│   ├── prompt1-spec-writer.md         # Spec generation prompt
│   ├── prompt2-spec-auditor.md        # Spec audit prompt
│   ├── prompt3-post-audit.md          # Implementation audit prompt
│   └── implementation-primer.md       # Implementation constraints & safeguards
├── schemas/
│   ├── spec-audit-verdict.json        # Structured output: spec verdicts
│   └── impl-audit-verdict.json        # Structured output: code verdicts
├── templates/
│   ├── config.yaml                    # Pipeline configuration template
│   └── PIPELINE-SPEC.md              # Operational runbook template
└── examples/
    └── working-doc-example.md         # Example working doc input
```

---

## How it works

### The working doc (your input)
A plain-English description of what you want built. The more specific you are about *what* to build, *what NOT to change*, and *edge cases*, the better the output. See `examples/working-doc-example.md` for format.

There's also an optional interactive helper (`prompts/prompt0-working-doc.md`) — give it to a coding agent and it'll walk you through creating a solid working doc with codebase discovery, impact analysis, and design decisions.

### The pipeline (what your clawbot runs)

**Phase 1-2: Spec generation.** A coding agent reads your codebase + working doc and produces an implementation spec. The spec prompt structurally prevents every common LLM failure mode — drift, incompleteness, pattern invention, missing error handling.

**Phase 3: Spec audit loop.** A *separate* coding agent session audits the spec against the working doc and actual codebase. Produces a structured JSON verdict: `IMPLEMENTABLE`, `NOT_IMPLEMENTABLE_YET`, or `ESCALATE`. If not implementable, your clawbot extracts the issue ledger, feeds it to the spec writer for targeted fixes, then re-audits. Up to N iterations, converging via the ledger system.

**Phase 4-4.5: Implementation + build verification.** A coding agent implements the spec. Build and lint run automatically. If they fail, the agent gets the errors and fixes them.

**Phase 5: Code audit loop.** Another separate session audits the actual implementation against the spec. Same convergence system — structured verdicts, issue ledgers, targeted fixes, re-audits. Catches spec deviations, bugs, missing edge cases, security issues.

**Phase 6: Finalize.** Commit, push, generate summary report, notify you. You review the branch and merge.

### The verdict system
Audits produce machine-parseable JSON with severity tiers (`critical` / `major` / `minor` / `info`). Only critical and major block the pipeline. Each issue gets a stable ID that persists across re-audits, enabling convergence tracking. When max iterations are hit: `blocker_count = 0` auto-promotes, `blocker_count > 0` escalates to you.

### Recovery
Everything is on disk. If your clawbot's context resets at any point, the next session reads `state.json`, checks the `resume_hint`, and resumes from exactly where it left off. Wire it into your clawbot's heartbeat and recovery is automatic.

---

## What you need

- **A clawbot** — any persistent AI agent that can run shell commands (OpenClaw, or your preferred orchestrator)
- **A coding agent CLI** — [Codex CLI](https://github.com/openai/codex), [Claude Code](https://github.com/anthropics/claude-code), [Aider](https://github.com/paul-gauthier/aider), or anything with autonomous mode + repo access + structured output
- **Git** + **jq** + your project's build toolchain

Built with OpenClaw + Codex CLI, but the framework is **agent-agnostic**. The pipeline logic works with any orchestrator + any coding agent that supports: (1) autonomous execution, (2) read/write repo access, (3) structured JSON output, and (4) session resume.

---

## Adapt it to your stack

The pipeline is language-agnostic and project-agnostic. What you customize:

| What | Where |
|------|-------|
| Build/lint/test commands | `templates/config.yaml` |
| Base branch, branch prefix | `templates/config.yaml` |
| Sandbox restrictions, env quirks | `prompts/implementation-primer.md` |
| Coding agent CLI commands | Shell commands in `docs/framework.md` Section 7 |

Works with Node, Python, Rust, Go, Java — anything with a build step. The core architecture is universal:
- Phase-gate pipeline (spec → audit → implement → audit → ship)
- Issue ledger convergence (the death spiral fix)
- Structured verdict schemas with severity gating
- Disk-based state management
- Heartbeat recovery

---

## Lessons from production

Battle-tested on a real codebase. Save yourself the debugging time.

1. **`exec resume` doesn't support `-o` or `--output-schema`.** Include schemas inline and tell the agent to write files via shell commands. This will bite you first.

2. **Without issue ledgers, audits death-spiral.** The auditor finds new tangential issues every round instead of verifying fixes. The ledger system fixes this. Biggest lesson learned.

3. **Always demand the complete updated spec on revisions.** If you say "fix issues," the AI returns a patch. Explicitly demand the full end-to-end spec every time.

4. **Codify your environment constraints.** If port binding is blocked or certain commands fail in your sandbox, put it in `implementation-primer.md`. Otherwise the coding agent discovers it the hard way — wasting an entire iteration.

5. **State is king.** If it's not in `state.json`, it didn't happen. Your future self (after a context reset) depends entirely on what's on disk.

6. **First run will have issues.** That's expected. Update your primer and runbook after each run. It gets better fast.

---

## Origin

Built by [@OlypsisAli](https://github.com/OlypsisAli) while shipping features on [Thesis](https://thesis.so), Feb 2026.

Born out of frustration with the gap between "AI can write code" and "AI can ship production features." ClawCode is the bridge.

---

## License

MIT — use it, fork it, adapt it, ship with it.
