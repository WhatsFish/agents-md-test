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
