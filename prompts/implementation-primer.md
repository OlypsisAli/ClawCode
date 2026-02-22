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

> **CUSTOMIZE THESE FOR YOUR PROJECT.** These are examples — replace with your actual constraints.

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
