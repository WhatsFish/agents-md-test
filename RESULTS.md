# Empirical Test Results: How Agents Load Repo Knowledge

Tested 2026-06-09 against **GitHub Copilot CLI 1.0.60** and **Claude Code 2.1.169** (Opus 4.8 / 1M).

Methodology: each mechanism gets a distinct canary codeword in a dedicated file; agents are asked to (a) reveal the codeword and (b) self-report how it reached their context (auto-injected at startup vs read-on-demand vs not loaded).

## Master matrix

| Mechanism | File path | Copilot CLI | Claude Code | Notes |
|---|---|:---:|:---:|---|
| OpenAI "agents.md" standard | `AGENTS.md` (root) | ✅ auto-inject | 🟡 read-on-demand | Claude sees the filename and proactively reads it, but it is not injected at startup |
| Nested `AGENTS.md` | `<subdir>/AGENTS.md` | ✅ auto-inject + merged with root | ❌ never automatic | Even when at cwd, Claude does not auto-load nested files |
| Anthropic project instructions | `CLAUDE.md` (root) | ✅ auto-inject | ✅ auto-inject | Both treat it as a primary source |
| GitHub Copilot project instructions | `.github/copilot-instructions.md` | ✅ auto-inject | ❌ ignored | The official Copilot location for the GitHub web UI / Coding Agent / IDE |
| Glob-scoped instructions | `.github/instructions/*.instructions.md` with `applyTo:` frontmatter | ✅ **listed at startup, lazy-loaded on glob match** | ❌ ignored | Copilot advertises the file at startup with its `applyTo` glob, then pulls full content when a matching file enters context |
| Gemini CLI instructions | `GEMINI.md` | ❌ not auto-loaded¹ | ❌ ignored | Despite being listed in Copilot's `/help` text, our run did not auto-inject it |
| Anthropic global user instructions | `~/CLAUDE.md` | ❌ ignored | ✅ auto-inject | The "personal preferences across all repos" file for Claude |
| GitHub Copilot global user instructions | `~/.copilot/copilot-instructions.md` | ✅ auto-inject | ❌ ignored | The equivalent for Copilot CLI |
| Claude slash commands | `.claude/commands/<name>.md` | ❌ ignored | ✅ first-class (`/<name>`) | Frontmatter `description:` shown in `/help` |
| Claude Skills | `.claude/skills/<name>/SKILL.md` | ❌ ignored | ✅ auto-discovered | `description:` frontmatter is read at startup, skill body only when invoked (model decides based on user intent) |
| Claude subagents | `.claude/agents/<name>.md` | ❌ ignored | ✅ first-class via Task tool | Frontmatter `description:` and `tools:` define delegation |
| MCP servers (per-project) | `.mcp.json` | 🟡 via `--additional-mcp-config @.mcp.json` | ✅ auto-loaded | Copilot's per-project MCP autoload is weaker; the main config lives at `~/.copilot/mcp-config.json` |
| Forbidden-directory directive | rule in instructions file | ✅ honored | ✅ honored | Both refused to read `do-not-touch/SECRET.md` |

## Cross-vendor candidate files (added 2026-06-10)

A second batch of files was added to test whether **agents pick up rule files defined by *other* tools** purely by filename convention. Each file has a unique canary codeword (see [`CANDIDATES.md`](./CANDIDATES.md)). Tested with **Copilot CLI 1.0.60** via `copilot -p` in a fresh non-interactive session, asking the agent to recite codewords it already had in context without reading any files.

| File path | Convention owner | Copilot CLI | Claude Code |
|---|---|:---:|:---:|
| `CONTRIBUTING.md` | Universal repo convention | ❌ not auto-loaded | ❌ not auto-loaded |
| `CONVENTIONS.md` | Aider (`--read CONVENTIONS.md`) | ❌ not auto-loaded | ❌ not auto-loaded |
| `.cursorrules` | Cursor (legacy) | ❌ not auto-loaded | ❌ not auto-loaded |
| `.cursor/rules/main.mdc` | Cursor (modern, mdc format) | ❌ not auto-loaded | ❌ not auto-loaded |
| `.windsurfrules` | Windsurf | ❌ not auto-loaded | ❌ not auto-loaded |
| `.clinerules` | Cline | ❌ not auto-loaded | ❌ not auto-loaded |
| `.continue/rules.md` | Continue.dev | ❌ not auto-loaded | ❌ not auto-loaded |
| `.goosehints` | Block Goose | ❌ not auto-loaded | ❌ not auto-loaded |
| `.junie/guidelines.md` | JetBrains Junie | ❌ not auto-loaded | ❌ not auto-loaded |
| `.idx/airules.md` | Google Project IDX | ❌ not auto-loaded | ❌ not auto-loaded |
| `.roo/rules.md` | Roo Code (modern) | ❌ not auto-loaded | ❌ not auto-loaded |
| `.roorules` | Roo Code (legacy) | ❌ not auto-loaded | ❌ not auto-loaded |
| `INSTRUCTIONS.md` | Generic uppercase | ❌ not auto-loaded | ❌ not auto-loaded |
| `RULES.md` | Generic uppercase | ❌ not auto-loaded | ❌ not auto-loaded |
| `STYLEGUIDE.md` | Generic uppercase | ❌ not auto-loaded | ❌ not auto-loaded |

**Both agents — zero cross-vendor pickup.** The only auto-injected sources remain `AGENTS.md` / `CLAUDE.md` (plus, for Copilot, nested `AGENTS.md` and the lazy-listed `.github/instructions/*.instructions.md`). All 15 candidate files above are **reachable on demand** (Test 4 in the CONTRIBUTING/RULES deep-dive below shows both agents will open every one of them when asked to "list every codeword in the repo"), but no agent surfaces them automatically.

### Implications

- **Don't rely on `CONTRIBUTING.md` to convey rules to AI agents.** Despite being the most universal "conventions" file in open source, no Copilot CLI auto-load happens.
- **Tool-specific files only matter to that tool.** `.cursorrules`, `.windsurfrules`, etc. are inert when the user runs a different agent. The convergence on `AGENTS.md` (OpenAI's cross-vendor proposal) is the only practical interop point today.
- **For broad agent reach, write `AGENTS.md` and symlink everything else to it.** `CLAUDE.md`, `GEMINI.md`, `.github/copilot-instructions.md` → all symlinks to `AGENTS.md`. This matches the existing recommendation in `STRUCTURE_GUIDE.md`.

¹ Copilot CLI's help text lists `GEMINI.md` as a loaded instruction file, but in our run it self-reported "I haven't read files like GEMINI.md from this working directory." Treat GEMINI.md support as unreliable for Copilot CLI as of 1.0.60.

## Behavioral nuances worth knowing

### Copilot CLI

- **Auto-injects everything up front** — even nested `AGENTS.md` files are merged into the system prompt at session start. Result: monorepo subprojects "just work" without prompting the agent to look around.
- **Lazy loads glob-scoped files** — `.github/instructions/*.instructions.md` are advertised at startup with their `applyTo` glob, but content is only loaded when the agent reads a matching file. This keeps token usage low.
- **Multiple sources merge cleanly** — root `AGENTS.md` + nested `AGENTS.md` + `~/.copilot/copilot-instructions.md` all coexist.

### Claude Code

- **Has a much richer `.claude/` ecosystem** — slash commands, skills, subagents, hooks, plugins, output styles. None of this has a Copilot equivalent in the same repo.
- **Only auto-injects two files**: `~/CLAUDE.md` (user) and the cwd's `CLAUDE.md` (project). Everything else (AGENTS.md, nested CLAUDE.md, `.github/instructions/`) is at best read on demand.
- **`AGENTS.md` read-on-demand is reliable** — Claude actively looks for the file name even though it isn't injected. But:
  - costs an extra Read tool call per session
  - the model might forget to look in deeply nested sessions
  - subdirectory AGENTS.md are usually missed entirely

### Both

- Honor "forbidden directory" / "don't touch X" rules from their loaded instructions file with high reliability.
- Cannot override a forbidden directive without explicit user permission.

## Deep-dive: CONTRIBUTING.md and RULES.md (2026-06-10)

A six-test matrix run against **Copilot CLI 1.0.60** *and* **Claude Code 2.1.169** specifically for `CONTRIBUTING.md` and `RULES.md`, varying (a) whether `AGENTS.md` mentions the files and (b) whether the user's question names the files explicitly. Each test was a fresh non-interactive session (`copilot -p` / `claude -p --dangerously-skip-permissions`) in the repo root with the agent instructed to self-report `[AUTO-INJECTED]` vs `[READ-ON-DEMAND]` and (in realistic tests) list the files it actually opened. Note: for Claude, `CLAUDE.md` is a symlink to `AGENTS.md`, so an "AGENTS.md mention" is automatically also a "CLAUDE.md mention."

### Copilot CLI

| # | Question style | AGENTS.md mentions? | Files read | Codewords retrieved? | Method |
|---|---|:---:|---|:---:|---|
| 1 | Direct: "What is the CONTRIBUTING/RULES codeword?" | no | CONTRIBUTING.md, RULES.md | ✅ both | `ls`+`cat` via shell |
| 2 | Direct: "What is the CONTRIBUTING/RULES codeword?" | yes | CONTRIBUTING.md, RULES.md | ✅ both | native `view` tool (cleaner) |
| 3 | "List every codeword in this repo" | yes | mentioned + nested AGENTS.md | ✅ 4 codewords | targeted `view` |
| 4 | "List every codeword in this repo" | no | **all ~22 files in repo** | ✅ all 18 codewords | broad shell grep, then `cat` |
| 5 | Realistic: "I'm a new contributor, brief me" | no | CONTRIBUTING, RULES, CONVENTIONS, STYLEGUIDE, INSTRUCTIONS, instructions/*, widget/AGENTS.md | ✅ all surfaced | `view` |
| 6 | Realistic: "I'm a new contributor, brief me" | yes | CONTRIBUTING, RULES, README, CONVENTIONS, STYLEGUIDE, instructions/* | ✅ all surfaced | `view` |

### Claude Code

| # | Question style | AGENTS.md/CLAUDE.md mentions? | Files read | Codewords retrieved? | Method |
|---|---|:---:|---|:---:|---|
| 1 | Direct: "What is the CONTRIBUTING/RULES codeword?" | no | CONTRIBUTING.md, RULES.md | ✅ both | native `Read` |
| 2 | Direct: "What is the CONTRIBUTING/RULES codeword?" | yes | CONTRIBUTING.md, RULES.md | ✅ both | native `Read` |
| 3 | "List every codeword in this repo" | yes | mentioned + nested AGENTS.md (4 files) | ✅ 4 codewords | targeted `Read` |
| 4 | "List every codeword in this repo" | no | **all ~22 files in repo** | ✅ all 21 codewords | broad `Glob`+`Read` |
| 5 | Realistic: "I'm a new contributor, brief me" | no | CONTRIBUTING, RULES, CONVENTIONS, STYLEGUIDE, INSTRUCTIONS, README, widget/AGENTS.md (re-read AGENTS too) | ✅ all surfaced | `Read` |
| 6 | Realistic: "I'm a new contributor, brief me" | yes | CONTRIBUTING, RULES, CONVENTIONS, STYLEGUIDE, widget/AGENTS.md | ✅ all surfaced | `Read` (5 files — tighter than Test 5) |

### What we learned

1. **CONTRIBUTING.md and RULES.md are never auto-injected** by either agent, even when AGENTS.md explicitly names them. They are not in the `custom_instruction`/system-prompt block at session start. This held in every test on both agents.

2. **Both agents reliably read-on-demand in two situations:**
   - **Filename appears in the user's question** (Tests 1, 2) → 100% read, with or without an AGENTS.md mention.
   - **Question semantically relates to "contributing" or "rules"** (Tests 5, 6) → both agents recognize the conventional filename and read it proactively even when AGENTS.md says nothing about it. Copilot CLI and Claude Code both have a learned prior that *"how do I contribute?"* → `CONTRIBUTING.md`.

3. **Mentioning the file in AGENTS.md (and therefore CLAUDE.md via symlink) has subtle, mostly cosmetic effects** for both agents:
   - Copilot: switches tool choice from `ls`+`cat` (Test 1) to native `view` (Test 2) — fewer tokens.
   - Claude: tightens which adjacent files get read on realistic questions (Test 5 read 8 files including a re-read of AGENTS/README; Test 6 read only 5, focused on the mentioned ones).
   - Neither agent promotes the mentioned file to auto-inject status.

4. **"List every codeword" (Test 4) triggers full-repo exploration on both agents** — every dot-prefixed tool-specific file (`.cursorrules`, `.windsurfrules`, `.clinerules`, `.goosehints`, `.junie/guidelines.md`, `.idx/airules.md`, `.roo/rules.md`, `.roorules`, `.continue/rules.md`, `.cursor/rules/main.mdc`) was opened and its codeword recited. Claude additionally pulled `.claude/{commands,agents,skills}/*` codewords (Copilot did too in its Test 4 run). So those files *are* reachable via on-demand reads; they are simply never auto-injected and never proactively read for normal questions.

5. **Agent-specific quirks worth noting:**
   - Copilot CLI auto-injects nested `packages/widget/AGENTS.md` even at root cwd (so `QUACK-...` is `[AUTO-INJECTED]` in Test 3); Claude does not (it's `[READ-ON-DEMAND]` for Claude in Test 3).
   - In realistic-question runs, Copilot tends to also crawl `.github/instructions/*.instructions.md`; Claude does not.
   - Claude is more chatty/structured in its briefing output (Test 5/6), explicitly distinguishing "real rules" vs "canary decoys."

### Practical recommendation

For rules you need an agent to follow **always**: put them directly in `AGENTS.md` (or symlinked equivalents like `CLAUDE.md` / `GEMINI.md` / `.github/copilot-instructions.md`). This is the only way to guarantee they reach the system prompt without a tool call.

For rules that are only relevant in specific contexts (contribution workflow, style guides): `CONTRIBUTING.md` and `RULES.md` are a *fine* place. Both Copilot CLI and Claude Code have filename priors and tool-use proactivity strong enough to surface them when the user's question implies they're needed. The cost is one extra read call per session, on either agent.

Mentioning these files from `AGENTS.md` is a minor polish — it makes the reads more deterministic and (for Copilot) uses the native `view` tool instead of shell `cat`. It is **not** required for either agent to find them.
