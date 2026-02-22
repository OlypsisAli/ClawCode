# Code Pipeline

**An automated pipeline that turns plain-English feature descriptions into fully implemented, audited, committed code — with minimal human intervention.**

You write a "working doc" describing what you want built. Your AI agent handles the rest: generates a spec, audits the spec, implements it, audits the implementation, fixes issues, and pushes a feature branch. You review and merge.

```
You write a Working Doc (plain English)
        |
  [Phase 1] Setup — branch created, state initialized
        |
  [Phase 2] Spec Generation — agent reads codebase + working doc, produces implementation spec
        |
  [Phase 3] Spec Audit Loop — separate agent audits the spec (up to N iterations, fixes between rounds)
        |
  [Phase 4] Implementation — agent implements the spec (writes actual code)
        |
  [Phase 4.5] Build Verification — run build + lint
        |
  [Phase 5] Post-Impl Audit Loop — separate agent audits the code (up to N iterations, fixes between rounds)
        |
  [Phase 6] Finalize — git commit, push, summary report
        |
  You review the feature branch and merge
```

---

## Why this exists

I got tired of the same failure modes every time I used AI to write code:

- **Spec drift** — the AI adds features you didn't ask for, "improves" your architecture, invents abstractions
- **Incomplete implementations** — 80% done, stubs everywhere, TODOs that never get resolved
- **Audit death spirals** — you ask the AI to review its own work, it finds new tangential issues each round, the loop never converges
- **Context loss** — the AI forgets what it was doing mid-implementation and starts from scratch
- **No accountability** — nothing is tracked, no artifacts are saved, you can't tell what happened or why

This pipeline fixes all of those. The key ideas:

1. **The orchestrator doesn't write code.** One AI manages the process (phases, state, escalation). A separate coding agent writes the actual code. This prevents the orchestrator from hallucinating code.

2. **Issue ledgers force convergence.** After every audit, issues are extracted into a ledger. The ledger gets fed back to both the revision step AND the re-audit step. The re-audit can only verify those specific issues — no expanding scope, no new tangents. This is the single most important design decision in the pipeline.

3. **Everything is on disk.** State files, thread IDs, audit verdicts, issue ledgers. If the AI's context resets mid-pipeline, the next session reads `state.json` and picks up exactly where it left off.

---

## How to use it

**Give the framework doc to your AI agent.** That's it.

The file `docs/framework.md` is a complete, self-contained handoff document. It's written for an AI agent to read and execute. Hand it to your orchestrator (OpenClaw, Claude, ChatGPT, whatever you use as a persistent AI assistant) and say:

> "Read this. Set up the pipeline for my project. Here's my first working doc: [path]"

Your agent will:
1. Read the framework doc
2. Create the directory structure
3. Customize the config for your project
4. Run the pipeline on your working doc

The prompt files, schemas, and templates in this repo are pre-extracted for convenience — but they're also all contained in the framework doc itself.

---

## What's in this repo

```
code-pipeline/
├── README.md                          # You're here
├── docs/
│   └── framework.md                   # The complete framework (2000+ lines)
│                                      # This is the file you hand to your AI agent
├── prompts/
│   ├── prompt0-working-doc.md         # Interactive working doc creation helper
│   ├── prompt1-spec-writer.md         # Spec generation prompt
│   ├── prompt2-spec-auditor.md        # Spec audit prompt
│   ├── prompt3-post-audit.md          # Implementation audit prompt
│   └── implementation-primer.md       # Prepended to spec for implementation phase
├── schemas/
│   ├── spec-audit-verdict.json        # Structured output schema for spec audits
│   └── impl-audit-verdict.json        # Structured output schema for impl audits
├── templates/
│   ├── config.yaml                    # Pipeline configuration template
│   └── PIPELINE-SPEC.md              # Operational runbook template
└── examples/
    └── working-doc-example.md         # Example working doc input
```

---

## The pieces

### Working doc
A plain-English description of what you want built. This is your input. Write it however you want, but the more specific you are about *what* to build, *what not to change*, and *edge cases*, the better the output. See `examples/working-doc-example.md`.

There's also an optional interactive helper (`prompts/prompt0-working-doc.md`) you can give to a coding agent to help you think through and structure a working doc.

### Prompts
Five prompt files that drive the pipeline phases. The spec writer constrains LLM failure modes (drift, incompleteness, pattern invention). The auditors produce machine-parseable verdicts with stable issue IDs. The implementation primer includes codebase integrity safeguards that prevent the coding agent from scaffolding missing infrastructure.

### Schemas
JSON schemas for structured audit output. Spec audits produce `IMPLEMENTABLE` / `NOT_IMPLEMENTABLE_YET` verdicts with issue arrays. Impl audits produce `COMPLIANT` / `NOT_COMPLIANT` verdicts with separate compliance and bug arrays. Both include `blocker_count` for automated pass/fail gating.

### State management
Each feature gets a `state.json` that tracks phase, thread IDs, iteration counts, verdicts, and a `resume_hint` field — a one-sentence orientation for the next session. If your AI's context resets mid-pipeline, the next session reads `resume_hint` and knows immediately how to proceed.

### Issue ledger convergence
This is the core innovation. Without it, auditors find new tangential issues every iteration and the loop never converges (death spiral). With ledgers:
- After each audit, issues are extracted into a plain-text ledger
- The ledger is fed to the revision step (fix *exactly* these issues, nothing more)
- The ledger is fed to the re-audit step (verify *these specific issues*, don't expand scope)
- Re-audit marks each issue as `resolved`, `unresolved`, or `new`
- Only `new` issues are ones introduced by the revision — no scope creep

---

## What you need

- An AI agent that can run shell commands (your orchestrator)
- A coding agent CLI with autonomous mode — [Codex CLI](https://github.com/openai/codex), [Claude Code](https://github.com/anthropics/claude-code), [Aider](https://github.com/paul-gauthier/aider), or similar
- Git
- Your project's build toolchain
- `jq` for parsing JSON output

The framework is **agent-agnostic**. It was built with OpenClaw + Codex CLI, but the pipeline logic works with any orchestrator + any coding agent that supports: (1) autonomous mode, (2) repo access, (3) structured output, and (4) session resume.

---

## Adapting to your project

The pipeline is project-agnostic. What you customize:

| What | Where |
|------|-------|
| Build/lint/test commands | `templates/config.yaml` |
| Base branch, branch prefix | `templates/config.yaml` |
| Sandbox restrictions, environment quirks | `prompts/implementation-primer.md` |
| Coding agent CLI commands | Shell commands in `docs/framework.md` Section 7 |

The core that stays the same regardless of your stack:
- Phase-gate architecture (spec -> audit -> implement -> audit -> ship)
- Issue ledger convergence system
- Structured verdict schemas
- Disk-based state management
- Heartbeat recovery

---

## Lessons from production

These are from real runs. Save yourself the debugging time.

1. **`exec resume` doesn't support `-o` or `--output-schema`.** Include schemas inline and tell the agent to write files via shell commands. This will bite you first.

2. **Without issue ledgers, audits death-spiral.** The auditor finds new tangential issues every round instead of verifying fixes. The ledger system fixes this. This was the biggest lesson.

3. **Always demand the complete updated spec on revisions.** If you tell the spec writer to "fix issues," it returns a patch. Explicitly demand the full end-to-end spec every time.

4. **Environment constraints must be codified.** If port binding is blocked or certain commands fail in your sandbox, put that in `implementation-primer.md`. Otherwise the coding agent discovers it the hard way, wasting an iteration.

5. **State is king.** If it's not in `state.json`, it didn't happen. Your future self (after a context reset) depends entirely on what's on disk.

6. **First run will have issues.** Expect environment-specific surprises. Update your primer and runbook after each run.

---

## Origin

Built and battle-tested by [@OlypsisAli](https://github.com/OlypsisAli) on the [Thesis](https://thesis.so) project, Feb 2026.

---

## License

MIT
