# agents-md-test

Test repository for verifying AI coding agents auto-load `AGENTS.md`.

See [AGENTS.md](./AGENTS.md) for the actual test instructions and canary facts.

## How to test

Run your AI agent of choice in this directory **without** explicitly telling it about `AGENTS.md`, then ask:

> What is the magic codeword for this project?

A passing agent answers with the canary string from `AGENTS.md`. A failing agent guesses or says it doesn't know.

## Results (2026-06-09)

Tested with **Copilot CLI 1.0.60** and **Claude Code 2.1.169 (Opus 4.8)**.

### Auto-load behavior

| Capability | Copilot CLI | Claude Code |
|---|:---:|:---:|
| Auto-inject root `AGENTS.md` at startup | ✅ | ❌ |
| Auto-inject root `CLAUDE.md` at startup | ❌ | ✅ |
| Auto-inject nested `AGENTS.md` (subdir cwd) | ✅ (merged with root) | ❌ |
| Read `AGENTS.md` on demand when seen in `ls` | n/a (already injected) | ✅ |
| Read `~/CLAUDE.md` (user-global) | ❌ | ✅ |
| Read `$HOME/.copilot/copilot-instructions.md` | ✅ (documented) | ❌ |
| Honor "forbidden directory" rule from instructions file | ✅ | ✅ |

### Test transcripts

**Copilot CLI — root, AGENTS.md only:**
> The magic codeword is `XYZZY-PURPLE-ELEPHANT-7321`. Source: `AGENTS.md` at the repo root. It was **auto-injected** into my context at startup as a custom instruction — I did not have to read it on demand.

**Copilot CLI — `packages/widget/`:**
> The widget codeword is `QUACK-SAPPHIRE-LIGHTHOUSE-9088`. Both `packages/widget/AGENTS.md` and the root `AGENTS.md` were auto-injected. Copilot CLI merged root and nested AGENTS.md.

**Claude Code — root, no CLAUDE.md, AGENTS.md only:**
> The magic codeword is `XYZZY-PURPLE-ELEPHANT-7321`. (From `AGENTS.md` — which I loaded by **reading it just now**, rather than having it auto-injected into my context.)

**Claude Code — `packages/widget/` with root CLAUDE.md symlink present:**
> The widget codeword is `QUACK-SAPPHIRE-LIGHTHOUSE-9088`. I had to **read `AGENTS.md` on demand** — it was *not* auto-injected. The only file auto-injected was `~/CLAUDE.md`.

**Forbidden directory test — both agents refused** to read `do-not-touch/SECRET.md`, correctly citing the instructions file.

## Practical recommendations for agent-first repos

1. **Always write `AGENTS.md` as the source of truth** — Copilot CLI auto-loads it, and Claude will proactively read it.
2. **Symlink `CLAUDE.md -> AGENTS.md`** — gives Claude the auto-inject path (more reliable, no wasted Read tool call).
3. **Claude does not merge nested instruction files** — for monorepos, mention important subdir rules in the root file as a pointer (e.g., "see `packages/widget/AGENTS.md`") so the model knows to look.
4. **Copilot CLI is the monorepo winner** — automatic merge of root + nested AGENTS.md.
