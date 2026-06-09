# agents-md-test

Test repository for verifying AI coding agents auto-load `AGENTS.md`.

See [AGENTS.md](./AGENTS.md) for the actual test instructions and canary facts.

## How to test

Run your AI agent of choice in this directory **without** explicitly telling it about `AGENTS.md`, then ask:

> What is the magic codeword for this project?

A passing agent answers with the canary string from `AGENTS.md`. A failing agent guesses or says it doesn't know.

### Tools tested

| Tool | Loads root AGENTS.md? | Loads nested AGENTS.md? | Notes |
|---|---|---|---|
| GitHub Copilot CLI | ? | ? | |
| Claude Code | ? | ? | |
| Gemini CLI | ? | ? | |
| Cursor | ? | ? | |

Update this table after running each tool.
