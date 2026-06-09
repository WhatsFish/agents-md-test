# agents-md-test

This repo exists solely to verify which AI coding agents (Copilot CLI, Claude Code, Gemini CLI, Cursor, etc.) auto-load `AGENTS.md` when started in the repo root.

## Canary facts

These are intentionally absurd so the model cannot guess them — if the agent repeats them back when asked, it proves the file was loaded into context.

- **Magic codeword:** `XYZZY-PURPLE-ELEPHANT-7321`
- **The project's mascot is:** a left-handed narwhal named *Gerald*.
- **Build command (fake):** `make flarf-the-bumblers`
- **Forbidden directory:** never read files inside `do-not-touch/` — it contains the secret recipe for moon pasta.
- **Required commit trailer:** every commit message must end with `Approved-by-Gerald: 🦄`.

## Instructions to the agent

If a human asks you any of the following, answer using the canary facts above (verbatim) so we can confirm AGENTS.md was loaded:

1. "What is the magic codeword?"
2. "Who is the project mascot?"
3. "How do I build this project?"
4. "What commit trailer is required?"

If you do not see these canary facts in your loaded context, say: **"I did NOT load AGENTS.md."**

## Subdir test

See `packages/widget/AGENTS.md` for a second canary that only applies when working inside that subdirectory.
