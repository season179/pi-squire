# codex-plugin-cc Reference

Source: https://github.com/openai/codex-plugin-cc

This file captures what codex-plugin-cc does and how it is built, as background for the sibling plugin we are designing here (`pi-squire`). Direction is intentionally inverse: codex-plugin-cc lets **Claude Code delegate to Codex**; we are building **Codex delegating to Pi**. The patterns transfer; the specifics do not.

## Purpose

Integrates the OpenAI Codex CLI into a Claude Code session so the Claude agent can run automated reviews, adversarial pressure-testing, and task delegation against Codex without leaving the terminal. Operations run either in foreground or as background jobs.

Transport/runtime boundary: the plugin sits between Claude Code's environment and the local Codex CLI / Codex app-server. The Codex CLI is wrapped, not extended.

## Top-level layout

```
.claude-plugin/
  marketplace.json                  # marketplace registration
plugins/codex/
  .claude-plugin/plugin.json        # plugin manifest (name=codex, current 1.0.4)
  agents/codex-rescue.md            # the codex:codex-rescue subagent
  commands/                         # markdown slash commands (/codex:review, /codex:rescue, ...)
  hooks/hooks.json                  # SessionStart, SessionEnd, stop-review-gate
  prompts/                          # prompt templates (e.g. adversarial-review.md)
  scripts/codex-companion.mjs       # central dispatcher CLI
  scripts/lib/                      # helpers: args, codex, fs, git, process, prompts, state, jobs, render
  scripts/lib/app-server-broker.mjs # persistent session to Codex app-server
```

## Plugin manifest

`plugins/codex/.claude-plugin/plugin.json` — Claude Code plugin manifest with `name`, `version`, `description`, `author`. (Note: this is Claude Code's `.claude-plugin/`, not Codex's `.codex-plugin/`.)

`.claude-plugin/marketplace.json` — registers the plugin in the Claude Code marketplace as `openai-codex`, listing the `codex` plugin and pointing at its source path.

## Companion script (`codex-companion.mjs`)

JavaScript / Node. One `main()` entry point. Acts as a single dispatcher for every slash command and subagent call. Routes by argv to the runtime commands:

| Command              | Purpose                                        |
| -------------------- | ---------------------------------------------- |
| `setup`              | Verify Codex install + auth                    |
| `review`             | Standard Codex review                          |
| `adversarial-review` | Steerable adversarial review                   |
| `task`               | Task delegation (used by the rescue subagent)  |
| `status`             | Active + recent jobs                           |
| `result`             | Completed job output                           |
| `cancel`             | Terminate active background job                |

Lifecycle is wrapped by a single `runTrackedJob()` helper that creates the job record, runs the underlying Codex call, captures output, and updates state.

## Job tracking and state

State lives in a `state.json` file **inside the workspace root**. Each job carries an id, kind, title, workspace root, job class, summary, write flag, and status (`pending | running | completed | failed | cancelled`). Concurrency and stale recovery are managed by tracking status transitions and per-job files. The `state.mjs` module is the persistence boundary; everything else goes through it.

## App-server broker

`scripts/lib/app-server-broker.mjs` keeps a persistent connection to the Codex app-server over a Unix socket / named pipe. Without it, every command would re-launch the Codex CLI and pay the cold-start cost. The broker amortizes startup across many calls in the same Claude Code session.

This is the most "interesting" architectural choice in the plugin and the one that does *not* obviously transfer to a Pi delegation plugin (see below).

## Hooks

`hooks/hooks.json` registers:

- `SessionStart` and `SessionEnd` hooks (`session-lifecycle-hook.mjs`) — for warm/cold lifecycle.
- A `stop-review-gate-hook.mjs` — optional gate that blocks the Claude agent from stopping a turn if an outstanding review found unresolved issues. Enabled via `/codex:setup --enable-review-gate`.

Hooks consume Claude Code's stdin JSON payload (which exposes `session_id`, `transcript_path`, `cwd`, etc.).

## Skills used by the rescue subagent

Two named internal skills, each *thin*:

- **`codex-cli-runtime`** — dictates that the subagent uses *exactly one* `Bash` call to invoke `codex-companion.mjs <task>`. No exploration, no extra tool use.
- **`gpt-5-4-prompting`** — the subagent's only job, beyond routing, is to *tighten* the user's request into a better Codex prompt before forwarding. The skill explicitly forbids the subagent from inspecting the repo or reasoning independently.

The principle: keep the subagent as a thin forwarding wrapper; let the companion script and Codex itself do the work.

## MCP / tools

The plugin doesn't expose MCP tools to the agent. It uses Claude Code's built-in tools:

- `Bash(node:*)` to invoke `codex-companion.mjs`.
- `AskUserQuestion` to prompt the user (e.g. "continue previous rescue thread?").
- `Agent` to delegate to the `codex:codex-rescue` subagent.

So the *agent-facing surface* is: a slash command, a subagent, and a CLI script. No MCP server.

## Prompt templates

`prompts/adversarial-review.md` casts Codex as an "adversarial software reviewer" with structured fields: `task`, `operating_stance`, `attack_surface`, `review_method`, `finding_bar`, and a `structured_output_contract` so the result parses predictably.

The rescue path doesn't use a fixed template — instead the `gpt-5-4-prompting` skill rewrites the user's request into a Codex prompt at delegation time.

## Patterns worth lifting for `pi-squire`

| Pattern                                | Lift?         | Notes |
| -------------------------------------- | ------------- | ----- |
| Companion script as central dispatcher | **Yes**       | `pi-squire.mjs` already aligned. |
| Single `runTrackedJob()` lifecycle     | **Yes**       | Use the same shape; one wrapper per write-capable command. |
| `state.mjs` as the persistence boundary | **Yes**      | Keep state writes in one module so atomic-write + flock live in one place. |
| Thin forwarding subagent               | **Yes**       | If we later add a `pi-rescue` subagent, keep it forwarding-only. |
| "Tighten the prompt" skill            | **Maybe**     | Useful for quality, but our skill already teaches prompt contracts directly; revisit if delegated prompts are noticeably weak. |
| Workspace-root `state.json`            | **No**        | Decided: state lives at `~/.pi-squire/state/<workspace-hash>/`, outside both the user's repo and Codex's plugin install dir, so plugin updates can't wipe it. |
| App-server broker                      | **No**        | `acpx` already provides persistent named sessions; the broker would re-implement what `acpx -s <name>` already gives us. |
| `SessionStart` / `SessionEnd` hooks    | **Defer**     | Codex's hook surface differs; not needed for MVP. |
| Stop-review-gate hook                  | **Defer**     | Spec already lists "automatic final-review gate" as a deferred item. |
| Slash commands as user surface         | **Yes**       | The Codex skill can document equivalent invocations for humans (`pi-squire review`, etc.) — the agent reaches them through the skill, the human can still type them directly. |
| Adversarial-review structured prompt   | **Yes**       | Use as the basis for `pi-squire challenge`'s prompt template. |

## Patterns intentionally diverged

- **No MCP server** in either plugin (codex-plugin-cc uses Claude Code built-ins; pi-squire uses Codex skill + CLI).
- **Different transport**: codex-plugin-cc shells to the Codex CLI / app-server; pi-squire shells to `acpx`. `acpx` is already a structured ACP client, so we get persistent sessions, named sessions, prompt queueing, cancel, status, JSON output — all things codex-plugin-cc had to build itself.
- **Direction**: this matters for prompts and skill descriptions. codex-plugin-cc's `gpt-5-4-prompting` is tuned for Codex-as-receiver; pi-squire's prompts are tuned for Pi-as-receiver and are simpler because the host (Codex) is the one doing the heavy lifting.
