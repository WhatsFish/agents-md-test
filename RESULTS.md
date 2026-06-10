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
| `CONTRIBUTING.md` | Universal repo convention | ❌ not auto-loaded | (untested) |
| `CONVENTIONS.md` | Aider (`--read CONVENTIONS.md`) | ❌ not auto-loaded | (untested) |
| `.cursorrules` | Cursor (legacy) | ❌ not auto-loaded | (untested) |
| `.cursor/rules/main.mdc` | Cursor (modern, mdc format) | ❌ not auto-loaded | (untested) |
| `.windsurfrules` | Windsurf | ❌ not auto-loaded | (untested) |
| `.clinerules` | Cline | ❌ not auto-loaded | (untested) |
| `.continue/rules.md` | Continue.dev | ❌ not auto-loaded | (untested) |
| `.goosehints` | Block Goose | ❌ not auto-loaded | (untested) |
| `.junie/guidelines.md` | JetBrains Junie | ❌ not auto-loaded | (untested) |
| `.idx/airules.md` | Google Project IDX | ❌ not auto-loaded | (untested) |
| `.roo/rules.md` | Roo Code (modern) | ❌ not auto-loaded | (untested) |
| `.roorules` | Roo Code (legacy) | ❌ not auto-loaded | (untested) |
| `INSTRUCTIONS.md` | Generic uppercase | ❌ not auto-loaded | (untested) |
| `RULES.md` | Generic uppercase | ❌ not auto-loaded | (untested) |
| `STYLEGUIDE.md` | Generic uppercase | ❌ not auto-loaded | (untested) |

**Copilot CLI verdict:** zero cross-vendor pickup. The only auto-injected sources remain `AGENTS.md` (root + nested) plus the lazy-listed `.github/instructions/*.instructions.md` files. When asked to list files in its system prompt, the agent self-reported only `AGENTS.md` and the two `.instructions.md` files — no mention of `.github/copilot-instructions.md` (likely deduplicated because it is symlink/duplicate of `AGENTS.md`).

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
