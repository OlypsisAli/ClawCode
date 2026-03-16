<p align="center">
  <h1 align="center">ClawCode</h1>
</p>

<p align="center">
  <strong>A framework you hand to your clawbot so it can spec, audit, implement, and ship features by orchestrating Codex CLI sessions end-to-end.</strong>
</p>

<p align="center">
  <a href="#-quickstart">Quickstart</a> &nbsp;·&nbsp;
  <a href="#-how-it-works">How it works</a> &nbsp;·&nbsp;
  <a href="#-lessons-from-production">Lessons from production</a> &nbsp;·&nbsp;
  <a href="docs/framework.md">Full framework doc</a>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue" alt="MIT License" /></a>
  <a href="https://github.com/openai/codex"><img src="https://img.shields.io/badge/built_with-Codex_CLI-black" alt="Built with Codex CLI" /></a>
  <a href="https://github.com/openclaw/openclaw"><img src="https://img.shields.io/badge/agent-OpenClaw-purple" alt="OpenClaw" /></a>
</p>

---

> **Framework, not hosted runtime.** Point your [OpenClaw](https://github.com/openclaw/openclaw) bot at this repo and tell it:
>
> *"Read `docs/framework.md`. Set up the implementation pipeline for my project at `[your-repo-path]`."*
>
> ClawCode provides the runbook, prompts, schemas, and templates. You still need Git, jq, Codex CLI, and your project's normal toolchain installed locally.

```
  You write a working doc
         ↓
  ┌──────────────────────────────────────────────────────────────┐
  │  ClawCode Pipeline (your clawbot runs this)                  │
  │                                                              │
  │  Setup → Spec Gen → Spec Audit → Implement → Code Audit     │
  │    ↓         ↕              ↕                 ↕              │
  │ worktree   fix ↔ re-audit  build/lint      fix ↔ re-audit    │
  └──────────────────────────────────────────────────────────────┘
         ↓
  Feature branch ready for review
```

---

## 🚀 Quickstart

```
1.  Install or verify Git, jq, Codex CLI, and your repo's build toolchain
2.  Point your OpenClaw bot to this repo
3.  Tell it: "Read docs/framework.md. Set up the implementation pipeline for my project."
4.  Let it generate the pipeline directory, config, prompts, schemas, and runbook
5.  Write a working doc describing the feature you want built
6.  Tell it: "Run the pipeline on this working doc: [path]"
7.  Review the resulting feature branch
```

### Recommended stack

| Role | Tool | Details |
|------|------|---------|
| **Orchestrator** | [OpenClaw](https://github.com/openclaw/openclaw) | Manages phases, state, notifications, and escalation |
| **Coding agent** | [Codex CLI](https://github.com/openai/codex) | `codex exec` for fresh phases, `codex exec resume` for iterative audit/fix turns |
| **Model guidance** | `gpt-5.3-codex` as the stable baseline, `gpt-5.4` only if it proves reliable in your environment | Verify availability with a smoke test before every run; model support depends on your Codex auth flow and CLI version |

Current ClawCode guidance is built around modern Codex CLI behavior, not the old `codex-3.6-extra-high` era. In production, the most reliable path has been: keep Codex CLI updated, run the pipeline in an isolated worktree, extend stream retry/idle settings in `~/.codex/config.toml`, prefer `gpt-5.3-codex` for long pipeline runs, and treat `exec resume` as a continuity tool for iterative audits rather than a drop-in replacement for fresh `exec`.

---

## ⚙️ How it works

### Your clawbot is the orchestrator

OpenClaw handles phase transitions, launches Codex CLI sessions, tracks `state.json`, extracts issue ledgers, and escalates when the pipeline gets stuck. Separate Codex sessions do the actual code reading and writing. That separation is intentional: the orchestrator manages process; the coding agent manages code.

### The pipeline runs in a dedicated worktree

The production pattern is to create `.worktrees/pipeline` in Phase 1, run every Codex/build/lint command there, and remove it in Phase 6. That keeps the user's main checkout clean while still sharing the same Git object store and feature branch history.

### The pipeline

```
  Working Doc (you write this)
          |
    [1] Setup ──────────── feature branch, .worktrees/pipeline, state.json
          |
    [2] Spec Gen ───────── Codex reads codebase + working doc → implementation spec
          |
    [3] Spec Audit Loop ── audit → issue ledger → revise → re-audit
          |
    [4] Implementation ─── Codex implements the spec in the worktree
          |
    [4.5] Build Verify ─── run install/build/lint/test commands as configured
          |
    [5] Code Audit Loop ── audit code → issue ledger → fix → re-audit
          |
    [6] Finalize ───────── summary, push feature branch, clean up worktree
```

### Why the audit loops converge

ClawCode keeps the issue-ledger system, `blocker_count`, and `WITH_NOTES` auto-promotion logic intact:

- Each audit produces a stable issue ledger with IDs.
- Fix rounds are constrained to those issues only.
- Re-audits verify the same ledger and only admit truly new issues introduced by the fix.
- If max iterations are reached with `blocker_count = 0`, the pipeline auto-promotes to `IMPLEMENTABLE_WITH_NOTES` or `COMPLIANT_WITH_NOTES` and proceeds.

### Recovery is on disk

`state.json` tracks phase, thread IDs, iteration counts, verdicts, `resume_hint`, and `worktree_path`. If OpenClaw resets, the next session can read state, inspect artifacts, and resume with the right strategy instead of guessing.

---

## 🧠 Why this exists

The hard part of autonomous software delivery is not getting a model to emit code. The hard part is getting repeatable, reviewable, production-safe behavior:

- Specs drift unless prompts and audits are structured.
- Code audits sprawl unless issue scope is locked down.
- Long reasoning turns stall unless Codex CLI and stream settings are current.
- Recovery falls apart unless the pipeline writes state to disk and treats every phase as resumable.

ClawCode standardizes those controls so your clawbot can run a real implementation pipeline instead of a one-shot code generation prompt.

---

## 📦 What's in this repo

```
clawcode/
├── README.md
├── docs/
│   └── framework.md
├── prompts/
│   ├── prompt0-working-doc.md
│   ├── prompt1-spec-writer.md
│   ├── prompt2-spec-auditor.md
│   ├── prompt3-post-audit.md
│   └── implementation-primer.md
├── schemas/
│   ├── spec-audit-verdict.json
│   └── impl-audit-verdict.json
├── templates/
│   ├── config.yaml
│   └── PIPELINE-SPEC.md
└── examples/
    └── working-doc-example.md
```

`docs/framework.md` is the canonical setup and operations document. The prompt, schema, and template files are the extracted versions your clawbot copies into its own pipeline directory.

---

## 🩹 Lessons from production

1. **Check Codex CLI before every run.** Stale CLI versions and stale `~/.local/bin/codex` symlinks caused repeated stalls in production.
2. **Tune `~/.codex/config.toml` for long reasoning.** `stream_max_retries`, `stream_idle_timeout_ms`, `request_max_retries`, and `model_context_window` matter.
3. **Use an isolated worktree.** `.worktrees/pipeline` keeps the main checkout untouched and avoids unnecessary Git friction inside agent turns.
4. **Use fresh `exec` and `resume` for different jobs.** Fresh `exec` is for new phases and stall recovery; `resume` is for iterative audits and targeted fix turns that need thread continuity.
5. **`exec resume` has sharp edges.** It does not support `-o` or `--output-schema`, so re-audits must inline schemas and instruct Codex to write files itself.
6. **Issue ledgers stop audit death spirals.** Without them, every re-audit expands scope and never converges.
7. **`state.json` must be explicit.** If `worktree_path`, verdicts, and `resume_hint` are missing, recovery becomes guesswork.
8. **Milestone notifications should be proactive.** Send them as soon as the phase changes; do not bury them in heartbeat responses.

---

## License

MIT — use it, fork it, adapt it, ship with it.
