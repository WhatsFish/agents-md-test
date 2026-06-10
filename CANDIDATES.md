# Candidate instruction-file conventions under test

This document lists every "is this auto-loaded?" candidate file added to the repo, the agent/tool that nominally owns the convention, and the unique canary codeword inside the file.

Test protocol:

1. Start a fresh agent session in this directory (do **not** prompt it about these files).
2. Ask: *"Without reading any files now, list every magic codeword you already have in your auto-loaded context, and the source file for each."*
3. A codeword that appears in the answer ⇒ the file was auto-injected by that agent. A codeword that requires a tool call to retrieve ⇒ not auto-injected.

| File path | Convention owner | Canary codeword |
|---|---|---|
| `AGENTS.md` | OpenAI / cross-vendor "agents.md" standard | `XYZZY-PURPLE-ELEPHANT-7321` |
| `CLAUDE.md` → symlink to `AGENTS.md` | Anthropic Claude Code | (same as AGENTS.md) |
| `GEMINI.md` → symlink to `AGENTS.md` | Google Gemini CLI | (same as AGENTS.md) |
| `.github/copilot-instructions.md` | GitHub Copilot (all surfaces) | (shares `XYZZY-...` — same content as AGENTS.md) |
| `.github/instructions/python.instructions.md` | GitHub Copilot file-scoped | `PYTHON-MAGENTA-PARROT-4471` |
| `.github/instructions/typescript.instructions.md` | GitHub Copilot file-scoped | `TS-VIOLET-WALRUS-8826` |
| `packages/widget/AGENTS.md` | Nested AGENTS.md (monorepo) | `QUACK-SAPPHIRE-LIGHTHOUSE-9088` |
| `CONTRIBUTING.md` | Universal repo convention | `CONTRIB-EMERALD-MOOSE-1101` |
| `CONVENTIONS.md` | Aider (`--read CONVENTIONS.md`) | `CONV-CRIMSON-OWL-2202` |
| `.cursorrules` | Cursor (legacy) | `CURSOR-AMBER-FOX-3303` |
| `.cursor/rules/main.mdc` | Cursor (modern, mdc) | `CURSORNEW-VIOLET-OTTER-4404` |
| `.windsurfrules` | Windsurf | `WIND-INDIGO-WHALE-5505` |
| `.clinerules` | Cline | `CLINE-MAGENTA-BADGER-6606` |
| `.continue/rules.md` | Continue.dev | `CONT-TEAL-RACCOON-7707` |
| `.goosehints` | Block Goose | `GOOSE-MAROON-PUFFIN-8808` |
| `.junie/guidelines.md` | JetBrains Junie | `JUNIE-OCHRE-NEWT-9909` |
| `.idx/airules.md` | Google Project IDX | `IDX-CORAL-IBEX-1010` |
| `.roo/rules.md` | Roo Code (modern) | `ROO-PERIWINKLE-DINGO-1111` |
| `.roorules` | Roo Code (legacy) | `ROOLEG-FUCHSIA-OCELOT-1212` |
| `INSTRUCTIONS.md` | Generic | `INST-VERMILION-CAPYBARA-1313` |
| `RULES.md` | Generic | `RULES-CYAN-AXOLOTL-1414` |
| `STYLEGUIDE.md` | Generic | `STYLE-CHARTREUSE-ARMADILLO-1515` |

Results from each agent's run are recorded in [`RESULTS.md`](./RESULTS.md).
