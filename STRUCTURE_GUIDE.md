# Definitive Repo Structure Guide for AI Coding Agents

Practical, opinionated file layouts for three scenarios, based on the empirical results in [`RESULTS.md`](./RESULTS.md).

The unifying principle: **`AGENTS.md` is your single source of truth.** Every other agent-specific file is either a symlink to it or a small adapter that delegates to it.

---

## Scenario 1 — Copilot-only repo

You only use GitHub Copilot (CLI + web + Coding Agent). Keep it lean.

```
my-repo/
├── AGENTS.md                                # primary source of truth
├── .github/
│   ├── copilot-instructions.md  -> ../AGENTS.md   # symlink, for github.com UI
│   └── instructions/                        # optional: file-type-scoped rules
│       ├── python.instructions.md           # applyTo: "**/*.py"
│       ├── frontend.instructions.md         # applyTo: "web/**/*.{ts,tsx}"
│       └── tests.instructions.md            # applyTo: "**/*_test.py"
└── copilot-setup-steps.yml                  # only if using Copilot Coding Agent
```

**Why:**

- `AGENTS.md` is auto-injected by Copilot CLI.
- The `.github/copilot-instructions.md` symlink makes the same content visible to **GitHub.com web Copilot**, the **Copilot Coding Agent**, and **VS Code Copilot Chat**, which all look for that exact path. One source, four front-ends.
- `.github/instructions/*.instructions.md` is the right place for **file-type-scoped** rules (linting conventions, framework idioms, test-style expectations). They lazy-load only when relevant, so they don't bloat every session.
- `copilot-setup-steps.yml` preinstalls dependencies into the Coding Agent runner so it can `npm test` / `pytest` immediately.

**`AGENTS.md` skeleton:**

```markdown
# <Project> — Agent Guide

## Overview
One sentence on the project + tech stack.

## Build / test / lint
- Build: `<exact command>`
- Test:  `<exact command>`
- Lint:  `<exact command>`
- Typecheck: `<exact command>`

## Repo layout
- `src/` — production code
- `tests/` — pytest suite
- `scripts/` — operator tooling

## Conventions
- Style: ...
- Naming: ...
- Error handling: ...

## Workflow rules
- Commit messages: <format>
- Branch naming: <format>
- Never force-push, amend pushed commits, or rewrite shared history.

## Don't touch
- `generated/`
- `vendor/`
- `*.lock` unless explicitly told to upgrade

## Environment
- Required env vars: see `.env.example`
- Start dev server: `<command>`
```

---

## Scenario 2 — Claude + Copilot shared repo

Both agents will work in this repo, sometimes simultaneously (e.g., one user prefers Claude Code, another prefers Copilot CLI; or you delegate review to one and writes to the other).

```
my-repo/
├── AGENTS.md                                       # ★ single source of truth
├── CLAUDE.md                  -> AGENTS.md         # for Claude Code auto-inject
├── .github/
│   ├── copilot-instructions.md -> ../AGENTS.md    # for Copilot web/Coding Agent
│   └── instructions/                               # Copilot-only, file-type-scoped
│       └── *.instructions.md
└── .claude/                                        # Claude-only ecosystem
    ├── commands/                                   # /<name> slash commands
    │   ├── deploy.md
    │   └── review.md
    ├── skills/                                     # auto-discovered skills
    │   └── run-migration/
    │       ├── SKILL.md
    │       └── migrate.sh
    └── agents/                                     # subagents (Task tool)
        ├── code-reviewer.md
        └── test-writer.md
```

**Key decisions explained:**

| Decision | Reason |
|---|---|
| `CLAUDE.md` is a symlink to `AGENTS.md`, not a copy | Avoids drift. Claude auto-injects via the `CLAUDE.md` name; the content is `AGENTS.md`. |
| Don't symlink Claude-specific stuff into `.github/` | `.claude/commands/`, `.claude/skills/`, `.claude/agents/` are Claude-exclusive concepts that don't translate. Copilot will ignore them harmlessly. |
| Don't symlink `.github/instructions/` into `.claude/` | Claude has no glob-instruction concept. If a rule there is important for Claude too, lift it into `AGENTS.md`. |
| Keep `.gitignore` rules for `.claude/settings.local.json` | Per-developer Claude settings should not be committed. |

**Suggested `.gitignore` additions:**

```
# Claude per-user state (do not commit)
.claude/settings.local.json
.claude/projects/
.claude/sessions/

# Copilot per-user state (do not commit)
.copilot/session-state/
```

---

## Scenario 3 — Monorepo with subprojects, skills, and per-package guidance

A monorepo with multiple packages and shared/per-package skills. This is the most complex case and where the two agents diverge most.

```
my-monorepo/
├── AGENTS.md                                    # repo-wide rules + index
├── CLAUDE.md                  -> AGENTS.md
├── .github/copilot-instructions.md -> ../AGENTS.md
│
├── .claude/                                     # repo-wide Claude assets
│   ├── commands/
│   │   ├── new-package.md                       # /new-package <name>
│   │   └── release.md                           # /release <package>
│   ├── skills/
│   │   ├── changeset/                           # add a changeset entry
│   │   │   └── SKILL.md
│   │   └── db-migration/
│   │       ├── SKILL.md
│   │       └── migrate.sh
│   └── agents/
│       ├── monorepo-reviewer.md                 # cross-package review
│       └── upgrade-bot.md                       # bulk dep bumps
│
├── packages/
│   ├── api/
│   │   ├── AGENTS.md                            # ★ Copilot auto-merges this
│   │   ├── CLAUDE.md      -> AGENTS.md          # mostly for symmetry; Claude won't auto-load
│   │   └── src/
│   ├── web/
│   │   ├── AGENTS.md
│   │   ├── CLAUDE.md      -> AGENTS.md
│   │   └── .github/
│   │       └── instructions/
│   │           └── react.instructions.md        # applyTo: "**/*.tsx"
│   └── shared/
│       ├── AGENTS.md
│       └── CLAUDE.md      -> AGENTS.md
│
└── docs/
    └── agent-knowledge/                         # large background docs
        ├── architecture.md                      # linked from AGENTS.md, not auto-loaded
        ├── db-schema.md
        └── deploy-runbook.md
```

### How knowledge flows in a monorepo

```
                ┌──────────────────────────────┐
                │ Repo-wide rules (AGENTS.md)  │
                │  + index of subpackages      │
                │  + index of skills           │
                └──────────────────────────────┘
                           │
              ┌────────────┼─────────────┐
              ▼            ▼             ▼
      packages/api/   packages/web/  packages/shared/
       AGENTS.md      AGENTS.md      AGENTS.md
       (Copilot       (Copilot       (Copilot
        auto-merges)   auto-merges)   auto-merges)
```

### Workarounds for Claude's lack of nested auto-merge

Claude won't auto-load `packages/api/AGENTS.md`. Two options:

**Option A — explicit pointers in root (recommended):**

```markdown
# AGENTS.md

## Subpackage rules

When working inside one of these directories, **read its `AGENTS.md` first** before making changes:

- `packages/api/AGENTS.md` — backend conventions, DB access rules
- `packages/web/AGENTS.md` — React conventions, design system
- `packages/shared/AGENTS.md` — shared library export rules
```

Claude reliably honors these instructions and reads the named files. Costs one Read tool call per session per touched package.

**Option B — `.claude/commands/enter-package.md`:**

```markdown
---
description: Load the rules for a specific subpackage before working on it
---

The user is about to work in package: $ARGUMENTS

Read `packages/$ARGUMENTS/AGENTS.md` and apply its rules for the rest of this session.
```

Run `/enter-package api` at the start of a session. Less elegant but explicit.

### Skills vs Instructions vs Knowledge docs — when to use which

| Type | Lives in | When to use |
|---|---|---|
| **Instructions** (always-on rules) | `AGENTS.md`, `CLAUDE.md`, `.github/copilot-instructions.md` | Style, conventions, never-do rules, build commands. Short, declarative. |
| **Glob-scoped instructions** (file-type rules) | `.github/instructions/*.instructions.md` (Copilot only) | "When editing `*.test.ts`, use Vitest patterns." Loaded lazily. |
| **Skills** (procedures the agent runs) | `.claude/skills/<name>/SKILL.md` + scripts | "Run a DB migration", "Add a changeset", "Generate boilerplate". Multi-step, possibly with helper scripts. Auto-discovered by description match. |
| **Slash commands** (user-invoked shortcuts) | `.claude/commands/<name>.md` | "/deploy", "/review", "/triage". User-initiated; replaces verbose prompts. |
| **Subagents** (specialized roles via Task tool) | `.claude/agents/<name>.md` | "code-reviewer", "test-writer". Used when you want the main agent to delegate to a focused sub-context. |
| **Knowledge docs** (large background) | `docs/agent-knowledge/*.md` | Architecture, schemas, runbooks. **Link from `AGENTS.md`** so agents read on demand, not at startup. Avoids context bloat. |
| **MCP servers** (external tools) | `.mcp.json` (Claude) / `~/.copilot/mcp-config.json` (Copilot) | API integrations the agent should call (Linear, Sentry, internal services). |

### The "single source of truth" rule for knowledge

When the same rule applies to all agents, put it in `AGENTS.md`. When something is genuinely tool-specific (a Claude skill script, a Copilot glob rule), put it under the tool's directory **and reference it from `AGENTS.md`** so both agents know it exists, even if only one can execute it.

Example `AGENTS.md` snippet:

```markdown
## Available tooling

- Claude users: run `/deploy <env>` (defined in `.claude/commands/deploy.md`)
- Copilot users: run `bash scripts/deploy.sh <env>` (Copilot does not have slash commands)
- Both: see `docs/agent-knowledge/deploy-runbook.md` for the full process
```

This ensures parity even when the underlying invocation differs.

---

## Quick reference: per-file purpose cheat sheet

| File | Source of truth? | Read by Copilot | Read by Claude | Commit? |
|---|:---:|:---:|:---:|:---:|
| `AGENTS.md` | ✅ yes | auto-inject | on demand | ✅ |
| `CLAUDE.md` (symlink) | no | auto-inject | auto-inject | ✅ |
| `.github/copilot-instructions.md` (symlink) | no | auto-inject | ❌ | ✅ |
| `.github/instructions/*.instructions.md` | yes (file-type) | lazy on glob match | ❌ | ✅ |
| `.claude/commands/*.md` | yes | ❌ | slash command | ✅ |
| `.claude/skills/*/SKILL.md` | yes | ❌ | auto-discover | ✅ |
| `.claude/agents/*.md` | yes | ❌ | subagent via Task | ✅ |
| `.claude/settings.local.json` | per-user | ❌ | yes | ❌ |
| `docs/agent-knowledge/*.md` | yes (large) | linked, on demand | linked, on demand | ✅ |
| `~/CLAUDE.md` | yes (per-user) | ❌ | auto-inject | ❌ |
| `~/.copilot/copilot-instructions.md` | yes (per-user) | auto-inject | ❌ | ❌ |

---

## What to put where (decision tree)

```
Is the rule/knowledge...
│
├── Universal to this repo, all agents, every session?
│       → AGENTS.md (with CLAUDE.md + .github/copilot-instructions.md symlinks)
│
├── Specific to a subdirectory's code?
│       → <subdir>/AGENTS.md (+ optionally <subdir>/CLAUDE.md symlink)
│         and mention it from root AGENTS.md so Claude knows to read
│
├── Specific to a file glob (e.g., all *.test.ts)?
│       → .github/instructions/<x>.instructions.md (Copilot lazy-loads)
│         Claude users: lift to AGENTS.md if critical
│
├── A multi-step procedure (with helper scripts)?
│       → .claude/skills/<name>/SKILL.md (Claude)
│         + scripts/<name>.sh (Copilot users invoke via shell)
│         Document both in AGENTS.md
│
├── A user-invoked shortcut?
│       → .claude/commands/<name>.md (Claude only — Copilot has no equivalent)
│
├── A long reference document (architecture, schema, runbook)?
│       → docs/agent-knowledge/<topic>.md and LINK from AGENTS.md
│         (don't paste it into AGENTS.md — context bloat)
│
├── Personal to one developer?
│       → ~/CLAUDE.md (Claude) and ~/.copilot/copilot-instructions.md (Copilot)
│         + Copilot Memory user scope. DO NOT commit.
│
└── A secret / credential?
        → ~/.config/<project>.env (mode 600), referenced from AGENTS.md
          but never written to it. NEVER commit.
```
