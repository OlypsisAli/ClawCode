# ClawCode

### Give your clawbot a working doc. Get back a feature branch.

We've evolved in how we use AI to write software. First it was inline autocomplete. Then copilots. Then autonomous agents. We're now at the point where you should be able to hand an AI a feature description and have it go implement the whole thing. The capability exists — but the process doesn't. Models are smart enough to write the code, but capturing your intent precisely, preventing drift, making sure everything downstream actually works the way you expect — that's not an intelligence problem, that's a *process* problem.

ClawCode is a framework that wraps every step of feature implementation in structured prompts, audits, and convergence loops. You write a plain-English working doc. Your clawbot orchestrates the rest — spec generation, spec auditing, implementation, code auditing, and shipping a clean feature branch. You review and merge.

No SDK. No install. No dependencies. It's a single document your clawbot reads and executes.

```
  You write a Working Doc (plain English)
          |
    [1] Setup ──────────── branch created, state initialized
          |
    [2] Spec Gen ───────── reads codebase + working doc, produces implementation spec
          |
    [3] Spec Audit Loop ── audits the spec (up to N rounds, fixes between each)
          |
    [4] Implementation ─── implements the spec (writes actual code)
          |
    [4.5] Build Verify ─── run build + lint, fix if broken
          |
    [5] Code Audit Loop ── audits the code (up to N rounds, fixes between each)
          |
    [6] Finalize ───────── git commit, push, summary report
          |
    You review the feature branch and merge
```

---

## 🚀 Quickstart

```
1.  Clone this repo
2.  Open docs/framework.md
3.  Give it to your clawbot
4.  Say: "Read this. Set up the pipeline for my project."
5.  Write a working doc describing what you want built
6.  Say: "Run the pipeline on this working doc: [path]"
7.  Go do something else
8.  Come back to a feature branch ready for review
```

That's it. The framework doc is self-contained — your clawbot reads it, sets up the directory structure, customizes the config, and runs the pipeline. The prompt files, schemas, and templates in this repo are pre-extracted for convenience, but they're all inside the framework doc too.

> **Note:** ClawCode was built and optimized for [Codex CLI](https://github.com/openai/codex) as the coding agent. It works best with Codex because the pipeline relies on `codex exec` for autonomous execution, `--output-schema` for structured verdicts, and `exec resume` for session continuity across audit iterations. You can adapt it to other coding agents (Claude Code, Aider, etc.) but Codex is the path of least resistance.

---

## 🧠 The bigger picture

The way we build software with AI has gone through phases:

**Phase 1: Autocomplete.** AI suggests the next line. You accept or reject. It's a typing accelerator.

**Phase 2: Copilot.** AI writes functions, fills in blocks, answers questions inline. You're still driving — it's riding shotgun.

**Phase 3: Autonomous agents.** AI reads your codebase, writes code, runs commands. You describe what you want and it goes and builds it.

We're solidly in Phase 3 now. The models are capable enough. But there's a gap between "AI can write code" and "AI can ship a production feature" — and it's not about model intelligence. It's about process:

- How do you capture intent precisely enough that the AI doesn't reinterpret it?
- How do you make sure the spec actually covers edge cases and matches your data contracts before a single line is written?
- How do you audit the implementation without the audit loop spiraling into new tangential issues every round?
- How do you recover when the AI's context resets mid-implementation?

ClawCode is my answer. Structured prompts at every step. Machine-parseable audits with convergence guarantees. Disk-based state that survives anything. The AI does the work — the framework makes sure the work is right.

---

## 💥 Why I built this

I was shipping features on a complex codebase with AI agents and kept hitting the same walls:

**Spec drift** — you ask for one thing, the AI builds three things and "improves" your architecture along the way.

**The 80% problem** — implementation looks done until you notice the stubs, the TODOs, the empty error handlers, the missing edge cases.

**Audit death spirals** — you ask the AI to review its own work. It finds issues. You fix them. It finds *different* issues. You fix those. It finds *new* issues. The loop never converges. You've been going in circles for an hour.

**Context amnesia** — the AI forgets what it was doing mid-implementation. Starts over. Makes different decisions the second time. Nothing is consistent.

**Zero paper trail** — nothing is tracked. No artifacts are saved. When something goes wrong you're debugging blind.

I realized the fix wasn't a better model — it was a better process. Structured prompts and a framework around each step of implementation. ClawCode is that framework, standardized and open-sourced.

---

## ⚙️ How it works

### Your clawbot = the project manager

Your clawbot orchestrates the pipeline. It doesn't write code — it manages phases, launches Codex CLI sessions, parses structured verdicts, extracts issue ledgers, carries context between steps, and escalates when things go wrong. A separate Codex session does the actual code reading and writing. This separation prevents the orchestrator from hallucinating code and gives the coding agent full sandbox access.

### The pipeline

**Phase 1-2: Spec generation.** Codex reads your codebase + working doc and produces an implementation spec. The spec prompt structurally constrains every common LLM failure mode — drift, incompleteness, pattern invention, missing error handling.

**Phase 3: Spec audit loop.** A *separate* Codex session audits the spec against the working doc and actual codebase. Produces a structured JSON verdict: `IMPLEMENTABLE`, `NOT_IMPLEMENTABLE_YET`, or `ESCALATE`. If not implementable, your clawbot extracts the issue ledger, feeds it to the spec writer for targeted fixes, then re-audits. Up to N iterations, converging via the ledger system.

**Phase 4-4.5: Implementation + build verification.** Codex implements the spec end-to-end. Build and lint run automatically. If they fail, the agent gets the errors and fixes them.

**Phase 5: Code audit loop.** Another separate session audits the actual implementation against the spec. Same convergence system — structured verdicts, issue ledgers, targeted fixes, re-audits. Catches spec deviations, bugs, missing edge cases, security issues.

**Phase 6: Finalize.** Commit, push, generate summary report, notify you. You review the branch and merge.

### Issue ledger convergence

This is the core innovation — the thing that makes the audit loops actually work.

Without ledgers, auditors find new tangential issues every iteration and the loop never converges (death spiral). With ledgers:
- After each audit, issues are extracted into a plain-text ledger with stable IDs
- The ledger is fed to the fix step — fix *exactly these issues*, nothing more
- The ledger is fed to the re-audit step — verify *these specific issues*, don't expand scope
- Re-audit marks each as `resolved`, `unresolved`, or `new`
- `new` only counts if the fix *introduced* it — no scope creep, no tangents

The loop converges because both sides are constrained to the same issue set.

### Recovery

Everything is on disk. `state.json` tracks phase, thread IDs, iteration counts, verdicts, and a `resume_hint` — a one-sentence orientation for the next session. If your clawbot's context resets at any point (compaction, sleep, crash), the next session reads the state file and picks up exactly where it left off. Wire it into your clawbot's heartbeat and recovery is automatic.

---

## 📦 What's in this repo

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
│   ├── prompt3-post-audit.md         # Implementation audit prompt
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

## 🔧 What you need

- A **clawbot** — any persistent AI agent that can run shell commands (OpenClaw, or your preferred orchestrator)
- **[Codex CLI](https://github.com/openai/codex)** (recommended) — the coding agent that does the actual work. The pipeline is built around `codex exec`, `--output-schema`, and `exec resume`
- **Git** + **jq** + your project's build toolchain

> The framework is agent-agnostic at the architecture level — you *can* swap Codex for [Claude Code](https://github.com/anthropics/claude-code), [Aider](https://github.com/paul-gauthier/aider), or anything with autonomous mode + structured output + session resume. But it works best with Codex CLI out of the box.

---

## 🔀 Adapt it to your stack

The pipeline is language-agnostic and project-agnostic. What you customize:

| What | Where |
|------|-------|
| Build/lint/test commands | `templates/config.yaml` |
| Base branch, branch prefix | `templates/config.yaml` |
| Sandbox restrictions, env quirks | `prompts/implementation-primer.md` |
| Coding agent CLI commands | Shell commands in `docs/framework.md` Section 7 |

Works with Node, Python, Rust, Go, Java — anything with a build step. The core architecture is universal:
- Phase-gate pipeline (spec -> audit -> implement -> audit -> ship)
- Issue ledger convergence (the death spiral fix)
- Structured verdict schemas with severity gating
- Disk-based state management
- Heartbeat recovery

---

## 🩹 Lessons from production

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

Born out of the gap between "AI can write code" and "AI can ship production features." ClawCode is the bridge.

---

## License

MIT — use it, fork it, adapt it, ship with it.
