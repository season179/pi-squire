# pi-squire — Spec Grilling Log

This file captures the design conversation that has shaped `pi-delegation-plugin-spec.md` and `CONTEXT.md` so far. It is written so that a future Claude (or Codex, or you) can resume the grilling session cold and continue from the exact pending question.

## Project at a glance

- **Repo:** `pi-squire` at `/Users/season/Personal/pi-squire`.
- **Goal:** A Codex plugin that lets the Codex agent (Desktop or CLI) delegate scoped work to the Pi coding agent through the `acpx` CLI. Codex is the editor-in-chief; Pi is auxiliary.
- **Sister project (reference, not dependency):** `openai/codex-plugin-cc` — the inverse direction (Claude Code delegates to Codex). We documented it in `codex-plugin-cc-reference.md` and lifted several patterns.
- **External dependency:** `openclaw/acpx` (npm `acpx`, currently alpha at 0.6.1). Provides ACP client behavior, persistent named sessions, prompt queueing, cancel, status, JSON output, permission modes.

## Files in the workspace

| File                                    | Purpose                                                  |
| --------------------------------------- | -------------------------------------------------------- |
| `CONTEXT.md`                            | Glossary, relationships, flagged ambiguities for the domain. Updated inline as terms get resolved. |
| `pi-delegation-plugin-spec.md`          | The actual spec under design. Has been heavily edited during grilling. |
| `codex-plugin-cc-reference.md`          | Faithful technical summary of `openai/codex-plugin-cc` plus a per-pattern "lift / re-decide / skip" table. |
| `grilling-log.md` *(this file)*         | Session checkpoint. |

## Resolutions captured so far

In approximate order of when they were made. Every one of these has been written into `pi-delegation-plugin-spec.md` (Resolved Decisions section) and/or `CONTEXT.md`. This log restates them for narrative continuity.

1. **Pi Job semantics — first pass.** Originally `CONTEXT.md` defined a Pi Job as "one delegated run", but resume semantics implied multi-prompt. Resolved as: a Pi Job = one named `acpx` session = a multi-turn conversation. Pi Turn = one prompt within a job.

2. **Codex plugin shape verified.** Codex (Desktop + CLI) genuinely supports `.codex-plugin/plugin.json`, `skills/` with implicit-invocation descriptions, `.mcp.json`, hooks, and slash commands. Plugins live under `~/.codex/plugins/<name>/`. Desktop and CLI share the model.

3. **Host session id — investigated and dropped.** Codex does **not** expose a stable conversation id to MCP servers. Hooks do receive `session_id` via stdin JSON. Initial design (agent mints a UUID and passes `--session`) was specced, then *removed* later — see resolution 12.

4. **MCP dropped.** No MCP server in the plugin. The agent invokes `pi-squire` directly. Reasons: skills already drive proactive use, the runtime is naturally CLI-shaped, humans run the same commands the agent runs.

5. **Locking model.** Workspace key = canonical `git rev-parse --show-toplevel` (cwd fallback). Job-state JSON is the lock; mutations under `proper-lockfile` with 5s staleness ceiling. Stale-job recovery = pid-liveness + 24h hard TTL (Codex pushed back: pid reuse is real). Per-workspace scope across all host sessions. Override = `pi-squire task --force` with audit fields (`forced`, `forcedAt`); the skill forbids the agent from passing it.

6. **Long-running commands.** Foreground/blocking by default. Both Codex CLI (5min `background_terminal_max_timeout`) and Claude Code (10min `BASH_MAX_TIMEOUT_MS`) handle long shells natively. No in-runtime `wait` primitive. `--bg` is opt-in for fire-and-forget.

7. **Permissions.** One command = one posture. `brainstorm → --deny-all`; `review`/`challenge → --approve-reads --non-interactive-permissions fail`; `task → --approve-all`. Removed legacy `diagnosis` and `rescue` references. Removed the `--read` flag I had erroneously added to `brainstorm`.

8. **codex-plugin-cc reuse.** Documented in `codex-plugin-cc-reference.md`. Lifting: central-dispatcher script, `runTrackedJob()` lifecycle, `state.mjs` persistence boundary, thin forwarding subagent, adversarial-review prompt scaffold. Skipping: app-server broker (acpx already does persistent sessions). Deferring: hooks.

9. **State location.** Centralized at `~/.pi-squire/state/<workspace-hash>/` where `<workspace-hash> = sha256(canonical git toplevel || cwd)[:16]`. Outside `~/.codex/plugins/pi-squire/` (Codex's install dir) so plugin updates can't wipe state. Logs at `~/.pi-squire/state/<hash>/logs/<job-id>.log`, results at `~/.pi-squire/state/<hash>/results/<job-id>.txt`.

10. **Naming.** Plugin manifest name = `pi-squire`; slash commands = `/pi-squire:<verb>`; CLI binary = `pi-squire`; skill = `pi-delegation`. Reason: bare `pi` collides with the actual Pi coding agent; `pi-squire` is unambiguous.

11. **Resume facility — dropped.** User: "Pi is disposable. Codex/Claude are the important ones." So `resume-candidate` is removed. A Pi Job = one delegated subtask. Pi Jobs are not reused across delegations. The runtime auto-generates the `acpx` session name; the agent never picks or remembers it.

12. **Host session id — fully removed.** Downstream of 11. With no resume and disposable Pi, there's no remaining justification for tracking which Codex conversation created which Pi Job. `pi-squire status` returns all jobs in the workspace. The agent identifies its own jobs by job ids in its scrollback. `--session` flag removed from every command. `hostSessionId` removed from job records.

13. **`--write` flag for humans — not added.** Command name (`task` vs `review`/`challenge`/`brainstorm`) carries the write/non-write signal; the per-workspace write-lock + mandatory diff inspection + audit fields are the actual safety. Adding `--write` would be friction without new failure-mode coverage.

14. **Terminology cleanup.** "host agent" → "Primary Agent" everywhere in the spec (CONTEXT.md already pinned this). "host adapter" reference removed. Lowercase "primary agent" in prompt templates capitalized for consistency.

## The pending question (where the grill paused)

**Question 14 — Should we ship `/pi-squire:<verb>` slash commands?**

Codex plugins can ship slash commands as markdown files under `commands/`. codex-plugin-cc ships them (`/codex:review`, etc.). pi-squire's resolved naming implies they exist (`/pi-squire:<verb>`), but the plugin packaging section doesn't document them yet.

My recommendation when the user paused was:
- **Yes, ship them, but keep them thin.** Each slash command is a one-line markdown wrapper that shells out to `pi-squire <verb> $ARGS`. Behavior is identical to running the CLI directly. The skill remains the sole source of *when* to delegate; slash commands are pure invocation surface for humans.
- **MVP set:** `setup`, `review`, `challenge`, `brainstorm`, `task`, `status`, `result`, `cancel` (8 commands, 8 small markdown files).
- **Plugin packaging gains a `commands/` directory** alongside `skills/` and `scripts/`.

The user paused before answering. **Resuming should start with: "Are slash commands worth shipping in MVP, and if so, do you want all 8 or a subset?"**

## Likely follow-up branches still un-grilled

These came up implicitly during the session and weren't resolved. Worth touching after Q14:

- **`pi-squire setup` content.** What does it actually check? (Node version? `acpx` install? Pi adapter availability? Permission of state dir? Print config?). Spec is currently minimal.
- **`pi-squire doctor` command?** codex-plugin-cc-style health-check command for ongoing troubleshooting. Distinct from `setup` (one-time) and `paths` (just prints paths).
- **`acpx` versioning / pinning specifics.** Spec says "plugin package declares a tested acpx version range" but doesn't pin a version yet. Need to choose a baseline (acpx 0.6.x?) and decide enforcement (warn vs refuse).
- **Prompt templates polish.** Codex-plugin-cc's `adversarial-review.md` is highly structured (`task`, `operating_stance`, `attack_surface`, `review_method`, `finding_bar`, `structured_output_contract`). Our `challenge` prompt template is plainer; worth lifting their structure.
- **What does `host` get filled with?** The job record has `host: codex-desktop|codex-cli|claude-code|unknown`. The runtime needs a heuristic (env vars? parent process? config flag?). Spec just says "best-effort detection."
- **Are background workers detached how?** `task --bg` "spawns a detached worker." Implementation detail (`node --detached`? `nohup`? `setsid`?) hasn't been pinned.
- **No tests/MVP test plan yet.** The MVP scope lists features but not how they're verified.

## How to resume

1. Read `CONTEXT.md` first (small, current).
2. Read `pi-delegation-plugin-spec.md` (the spec; this is the design source of truth).
3. Read `codex-plugin-cc-reference.md` if you want context on what we're borrowing from.
4. Skim this file for chronology if anything in the spec seems surprising.
5. Pick up at Question 14 (slash commands), or pick a follow-up branch from the list above.

Skill that drove this session: `/grill-with-docs` from `/Users/season/.claude/skills/grill-with-docs`. Re-running it is a fine way to continue.

## Useful one-liners for resuming

```bash
# Check current state of all design docs
ls -la /Users/season/Personal/pi-squire/

# Diff the spec vs initial (if you commit before pausing)
git -C /Users/season/Personal/pi-squire diff HEAD -- pi-delegation-plugin-spec.md CONTEXT.md

# Verify acpx is still where we expect
which acpx 2>/dev/null; acpx --version 2>/dev/null

# Re-read codex-plugin-cc upstream (in case it has changed)
gh repo view openai/codex-plugin-cc
```

## Anchors to remember when resuming

- **Pi is disposable per subtask.** Don't accidentally re-introduce resume semantics, session-id management, or "continue prior Pi work" features. The user explicitly chose disposability.
- **No MCP.** Don't accidentally suggest exposing things as MCP tools — the agent invokes `pi-squire` via Bash, period.
- **Codex/Claude are smart enough.** The user's instinct on "agent harnesses can handle long shells natively" was right. Don't over-engineer wait/poll primitives.
- **Simple beats engineered.** When in doubt, the user prefers the simpler option.
