<p align="center">
  <h1 align="center">ClawCode</h1>
</p>

<p align="center">
  <strong>A framework you hand to your clawbot that allows it to spec, audit, implement, and ship any feature. Autonomously. No Hassle.</strong>
</p>

<p align="center">
  <a href="#-quickstart">Quickstart</a> &nbsp;·&nbsp;
  <a href="#-how-it-works">How it works</a> &nbsp;·&nbsp;
  <a href="#-the-bigger-picture">Why it exists</a> &nbsp;·&nbsp;
  <a href="docs/framework.md">Full framework doc</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-blue" alt="MIT License" />
  <img src="https://img.shields.io/badge/built_with-Codex_CLI-black" alt="Built with Codex CLI" />
  <img src="https://img.shields.io/badge/agent-OpenClaw-purple" alt="OpenClaw" />
</p>

---

> **Zero setup.** Point your [OpenClaw](https://openclaw.com) bot at this repo and tell it:
>
> *"Read `docs/framework.md`. Set up the implementation pipeline for my project at `[your-repo-path]`."*
>
> Your clawbot reads the framework, creates the directory structure, generates the config, and is ready to run. No install. No SDK. No dependencies.

```
  You write a working doc
         ↓
  ┌─────────────────────────────────────────────────────────┐
  │  ClawCode Pipeline (your clawbot runs this)             │
  │                                                         │
  │  Spec Gen → Spec Audit Loop → Implement → Code Audit   │
  │     ↑            ↕                            ↕         │
  │  codebase    fix ↔ re-audit              fix ↔ re-audit │
  └─────────────────────────────────────────────────────────┘
         ↓
  Feature branch ready for review
```

---

## 🚀 Quickstart

```
1.  Point your OpenClaw bot to this repo
2.  Tell it: "Read docs/framework.md. Set up the implementation pipeline for my project."
3.  Your clawbot sets up the pipeline automatically (directories, config, prompts, schemas)
4.  Write a working doc describing the feature you want built
5.  Tell it: "Run the pipeline on this working doc: [path]"
6.  Go do something else
7.  Come back to a feature branch ready for review
```

### Recommended stack

| Role | Tool | Details |
|------|------|---------|
| **Orchestrator** | [OpenClaw](https://openclaw.com) | Your clawbot — manages phases, carries context, escalates |
| **Coding agent** | [Codex CLI](https://github.com/openai/codex) | `codex exec` / `exec resume` — does the actual code reading & writing |
| **Model** | Codex 3.6 Extra High | The model ClawCode was built and tested with |

ClawCode was built with OpenClaw + Codex CLI running Codex 3.6 Extra High. This is the intended and tested configuration. The pipeline relies on `codex exec` for autonomous execution, `--output-schema` for structured audit verdicts, and `exec resume` for session continuity across iterations. You *can* adapt it to other agents, but this is the path of least resistance.

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

Your clawbot (OpenClaw) orchestrates the pipeline. It doesn't write code — it manages phases, launches Codex CLI sessions, parses structured verdicts, extracts issue ledgers, carries context between steps, and escalates when things go wrong. Separate Codex sessions do the actual code reading and writing. This separation prevents the orchestrator from hallucinating code and gives the coding agent full sandbox access.

### What your clawbot sets up

When you point your clawbot at `docs/framework.md`, it reads Section 4 (Step-by-Step Setup Checklist) and executes it automatically:

1. **Verifies prerequisites** — Git, jq, Codex CLI installed and authenticated
2. **Creates the pipeline directory** — `prompts/`, `schemas/`, `state/`, `logs/`, `inbox/`
3. **Generates `config.yaml`** — fills in your repo path, build commands, branch strategy, model
4. **Copies prompt and schema files** into place
5. **Generates `PIPELINE-SPEC.md`** — the operational runbook with your actual paths filled in
6. **Wires into its own memory** — adds pipeline awareness to AGENTS.md and heartbeat recovery to HEARTBEAT.md

After setup, your clawbot knows how to run the pipeline on any working doc you give it. It also knows how to resume a stalled pipeline after a context reset (via the heartbeat recovery system).

### The pipeline

```
  Working Doc (you write this)
          |
    [1] Setup ──────────── branch created, state.json initialized
          |
    [2] Spec Gen ───────── Codex reads codebase + working doc → implementation spec
          |
    [3] Spec Audit Loop ── Codex audits spec → verdict → fix → re-audit (up to N rounds)
          |
    [4] Implementation ─── Codex implements the spec → actual code
          |
    [4.5] Build Verify ─── build + lint → fix if broken
          |
    [5] Code Audit Loop ── Codex audits code → verdict → fix → re-audit (up to N rounds)
          |
    [6] Finalize ───────── commit, push, summary → you review and merge
```

**Phase 1-2: Spec generation.** Codex reads your codebase + working doc and produces an implementation spec. The spec prompt structurally constrains every common LLM failure mode — drift, incompleteness, pattern invention, missing error handling.

**Phase 3: Spec audit loop.** A *separate* Codex session audits the spec against the working doc and actual codebase. Produces a structured JSON verdict: `IMPLEMENTABLE`, `NOT_IMPLEMENTABLE_YET`, or `ESCALATE`. If not implementable, your clawbot extracts the issue ledger, feeds it to the spec writer for targeted fixes, then re-audits. Up to N iterations, converging via the ledger system.

**Phase 4-4.5: Implementation + build verification.** Codex implements the spec end-to-end. Build and lint run automatically. If they fail, the agent gets the errors and fixes them.

**Phase 5: Code audit loop.** Another separate session audits the actual implementation against the spec. Same convergence system — structured verdicts, issue ledgers, targeted fixes, re-audits. Catches spec deviations, bugs, missing edge cases, security issues.

**Phase 6: Finalize.** Commit, push, generate summary report, notify you. You review the branch and merge.

### 🔑 Issue ledger convergence

This is the core innovation — the thing that makes the audit loops actually work.

Without ledgers, auditors find new tangential issues every iteration and the loop never converges (death spiral). With ledgers:
- After each audit, issues are extracted into a plain-text ledger with stable IDs
- The ledger is fed to the fix step — fix *exactly these issues*, nothing more
- The ledger is fed to the re-audit step — verify *these specific issues*, don't expand scope
- Re-audit marks each as `resolved`, `unresolved`, or `new`
- `new` only counts if the fix *introduced* it — no scope creep, no tangents

The loop converges because both sides are constrained to the same issue set.

### 💾 Recovery

Everything is on disk. `state.json` tracks phase, thread IDs, iteration counts, verdicts, and a `resume_hint` — a one-sentence orientation for the next session. If your clawbot's context resets at any point (compaction, sleep, crash), the next session reads the state file and picks up exactly where it left off. Wire it into your clawbot's heartbeat and recovery is automatic.

---

## 📦 What's in this repo

```
clawcode/
├── README.md                          # You're here
├── docs/
│   └── framework.md                   # THE FILE — your clawbot reads this to bootstrap everything
│                                      #   Complete, self-contained, 2000+ lines
│                                      #   Setup checklist, phase commands, prompts, schemas — all in one
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

> **Note:** The prompt files, schemas, and templates are pre-extracted for reference. But `docs/framework.md` contains everything — your clawbot only needs that one file to set up and run the entire pipeline.

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
- Phase-gate pipeline (spec → audit → implement → audit → ship)
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
