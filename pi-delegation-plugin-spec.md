# Codex Pi Delegation Plugin Spec

Status: draft

## Goal

Build a Codex plugin that lets Codex delegate work to the Pi Coding Agent through `acpx`.

The Primary Agent can be either a Codex Desktop session or a Codex CLI session.

The Primary Agent remains the editor-in-chief. Pi is a cheaper auxiliary agent used for grunt work, independent diagnosis, brainstorming, rubber-ducking, and second-opinion review.

The plugin should encourage proactive use of Pi. The human should not need to say "ask Pi" for the Primary Agent to use it.

## Non-Goals

- Do not implement model selection in this plugin. Pi uses whatever model/provider the user has configured in Pi.
- Do not require a separate git worktree by default.
- Do not replace Codex as the final reviewer/integrator.
- Do not make Pi a native Codex subagent runtime. It is reached through plugin commands/tools backed by `acpx`.

## Architecture

Use a Codex plugin plus a small CLI runtime. The plugin ships **no MCP server** — the agent invokes the runtime CLI directly, guided by the skill.

```text
Codex Desktop thread  ─┐
                       ├─►  Codex plugin (skill: proactive policy + CLI usage)
Codex CLI conversation ─┘                  │
                                           ▼
                                pi-squire CLI runtime
                                           │
                                           ▼
                                       acpx CLI
                                           │
                                           ▼
                            Pi ACP adapter / Pi Coding Agent
```

`acpx` is the transport/runtime boundary. It already provides ACP client behavior, Pi adapter resolution, persistent sessions, named sessions, prompt queueing, cancel, status, permission modes, cwd scoping, and JSON/quiet output.

### Why no MCP?

The Codex agent can run shell commands directly, and Codex skills already drive proactive use through implicit invocation on the skill description. Wrapping a CLI runtime in an MCP server would add a long-lived process and a second protocol surface without unlocking new agent leverage. Dropping MCP also means humans run the same `pi-squire` commands the agent runs, which keeps debugging and dogfooding simple.

## Runtime

Provide a Node CLI:

```bash
node scripts/pi-squire.mjs <command> [options] [prompt]
# or, once installed via the Codex plugin, simply:
pi-squire <command> [options] [prompt]
```

Runtime requirements:

- Node compatible with the pinned `acpx` version.
- `acpx` installed globally or invoked through a pinned package command.
- Pi installed and configured by the user.
- No model flags are passed unless a future version explicitly adds that feature.

### Runtime Commands

```text
setup
review
challenge
brainstorm
task
status
result
cancel
```

### `setup`

Checks:

- Node is available.
- `acpx` is available.
- Pi is available enough for `acpx pi` to start.
- Optional: current workspace can create or inspect a named `acpx pi` session for a job.

Output:

- human-readable report by default
- JSON with `--json`

### `review`

Purpose: read-only second opinion on current work.

Behavior:

- Use `acpx --cwd <workspace> --approve-reads --non-interactive-permissions fail pi exec <prompt>`.
- Prompt Pi to inspect but not modify files.
- Prefer `--format quiet` for final user-facing output or `--format json --json-strict` when the runtime needs event parsing.
- Used for normal code review, branch review, or focused review.

### `challenge`

Purpose: adversarial review of approach, design assumptions, risks, and alternatives.

Behavior:

- Same permission posture as `review`.
- Prompt Pi to challenge the direction, not only implementation defects.
- Useful before shipping or before committing to an architecture.

### `brainstorm`

Purpose: cheap second brain without workspace mutation and without file inspection.

Behavior:

- Always run as `acpx --deny-all pi exec <prompt>`. The host supplies all context inline; Pi cannot read or write files.
- If the work needs Pi to inspect code, use `review` instead. There is intentionally no `--read` flag on `brainstorm`; one command, one permission posture.
- Output should be concise options, risks, or questions.

### `task`

Purpose: write-capable delegated work.

Behavior:

- Create the persistent named `acpx` session for this job (the runtime auto-generates the session name; see Session Model below):

```bash
acpx --cwd <workspace> --approve-all pi -s <job-session> <prompt>
```

- The Primary Agent (or human) chooses foreground vs background by passing or omitting `--bg`.
- The runtime records job state before launching.
- The runtime captures stdout/stderr or NDJSON event logs.
- The runtime snapshots git status before and after the run.

Default posture:

- `task` is write-capable.
- Use the same checkout by default.
- Do not require a worktree.
- Allow optional future `--worktree` mode for invasive or long-running work.

### `status`

Primary observability surface for both the agent and the human. Shows active and recent Pi Jobs for the current workspace.

`pi-squire status` (no flags) lists all jobs in the workspace, newest first. `pi-squire status --job <id>` returns the single matching record.

For each job the output includes:

- `id` (the runtime-assigned job id; what the agent and human use to reference the job)
- `kind` (`review` | `challenge` | `brainstorm` | `task`)
- `status` (`queued` | `running` | `completed` | `failed` | `canceled`)
- `staleness` (`null` | `pid-dead` | `ttl-exceeded`) — explains *why* a job was auto-transitioned to `failed`
- `foreground | background`
- `write` (boolean — was this a write-capable run?)
- `forced` (boolean — did the operator bypass the active-write check?)
- `workspaceRoot`
- `acpxSessionName` (auto-generated; surfaced for debugging only)
- `pid` when running
- `createdAt`, `startedAt`, `completedAt`
- `promptPreview` (short text)
- `errorPreview` (one-line summary when `status` is `failed`; full error in the log file)
- `logFile`, `resultFile` paths so the operator can `tail`/`cat` for full detail

`status` proactively reaps stale records on every call: any non-terminal record whose pid is dead or whose age exceeds the 24h TTL is transitioned to `failed` before output is rendered, with `staleness` set accordingly. This keeps observability honest without requiring a separate sweep command.

### `result`

Shows final stored output for a job.

Should include:

- final Pi output
- error output if failed
- `acpx` session identifiers when available
- touched/possibly touched files when known
- reminder that the Primary Agent must review before finalizing

### `cancel`

Cancels an active Pi job:

```bash
acpx --cwd <workspace> pi cancel -s <job-session>
```

If the runtime spawned a detached worker, also clean up local job state.

## Session Model

Each Pi Job is one named `acpx` session, scoped to **one delegated subtask**. Pi is treated as a disposable per-subtask helper: Pi Jobs are not reused across delegations, and there is no resume facility. The Primary Agent (Codex) is the only place where context accumulates across calls.

- A Pi Job maps one-to-one to a named `acpx` Pi session.
- A Pi Job usually has a single Pi Turn; a Pi Job may naturally have a few internal Turns if Pi asks a clarifying question and continues — but each new delegation starts a fresh Pi Job, never reuses a previous one.
- The runtime auto-generates the `acpx` session name per Pi Job; the agent never has to choose or remember it.
- The runtime does **not** track which Host Session created which Pi Job. `pi-squire status` returns all Pi Jobs in the workspace; the agent identifies its own jobs by the job ids in its scrollback. Two Codex windows on the same workspace will see each other's jobs in `status` output — that's accepted noise in exchange for a simpler model.

## Job State

State is centralized, not stored inside the user's workspace. This keeps repos clean, supports read-only checkouts, and survives `git clean -fdx`.

Layout:

```text
~/.pi-squire/state/<workspace-hash>/
  jobs.json        # canonical job records (atomic write under proper-lockfile)
  logs/<job-id>.log
  results/<job-id>.txt
```

- `<workspace-hash>` is `sha256(canonical absolute path)[:16]`, where the canonical path is `git rev-parse --show-toplevel` resolved through any symlinks, falling back to the canonical absolute `--cwd` when not in a git repository.
- Distinct git worktrees of the same repo therefore get distinct hashes and run independently.
- A `pi-squire paths` command prints the resolved state directory, log directory, and results directory for the current workspace, for debugging.

XDG compliance is intentionally skipped for v1 — the plugin keeps its state under Codex's own plugin data root because that matches the host's convention. Users who care can symlink.

Each job record:

```json
{
  "id": "pi-task-...",
  "host": "codex-desktop|codex-cli|claude-code|unknown",
  "kind": "review|challenge|brainstorm|task",
  "workspaceRoot": "/abs/path",
  "cwd": "/abs/path",
  "acpxSessionName": "pi-squire-<short-uuid>",
  "status": "queued|running|completed|failed|canceled",
  "staleness": null,
  "pid": 12345,
  "promptPreview": "short text",
  "errorPreview": null,
  "write": true,
  "forced": false,
  "forcedAt": null,
  "background": false,
  "createdAt": "...",
  "startedAt": "...",
  "completedAt": "...",
  "logFile": "...",
  "resultFile": "...",
  "beforeGitStatus": "...",
  "afterGitStatus": "...",
  "touchedFiles": []
}
```

Notes:

- `host` is best-effort detection from environment hints; defaults to `unknown` when the runtime cannot tell which agent harness invoked it.
- `acpxSessionName` is auto-generated by the runtime — the agent never picks it. It's stored only for debugging (`acpx sessions show`).
- `staleness` is `null` until the runtime auto-transitions the record to `failed` because of pid death (`pid-dead`) or 24h TTL (`ttl-exceeded`).
- `errorPreview` is a one-line summary; the full error stack/output lives in `logFile`.
- `forced` records whether the operator bypassed the active-write check via `--force`.

Keep the newest 50 jobs per workspace by default.

## Same-Checkout Safety Model

Same checkout is the default because this should feel ergonomic, direct, and integrated into the current Codex workflow.

Safety comes from mode selection, job tracking, and host review, not mandatory checkout isolation.

### Rules

- Allow multiple read-only Pi jobs in parallel (no lock taken).
- Allow only one Pi write job per workspace at a time, **across all host sessions** sharing that checkout.
- Do not start a second Pi write job while one is active unless the human explicitly overrides.
- The Primary Agent should avoid editing the same files while a Pi write job is active.
- After a write-capable Pi job, the host must inspect the diff before claiming completion.
- The host must run or request appropriate verification before finalizing.

### Enforcement

- **Workspace key** is the canonical absolute path of `git rev-parse --show-toplevel`, falling back to the canonical absolute `--cwd` when not inside a git repository. Distinct git worktrees get distinct keys and run independently.
- **Lock mechanism** is the job-state JSON store itself; there is no separate lockfile per job. Before starting a write job, the runtime opens the per-workspace state file under a `proper-lockfile` flock, performs read → check → write atomically, then releases. The critical section must complete entirely under the flock; releasing the flock before the new record is durable allows two starters to both believe they have the lock.
- **Lockfile staleness** is set to ~5 seconds because the critical section is a single JSON read-modify-write. A crash between acquire and write therefore self-heals quickly.
- **Stale-job recovery** uses pid-liveness as the primary signal *plus* a 24h hard TTL as a safety net. Pid reuse is a real failure mode (Linux pids wrap), so a non-terminal record whose recorded pid is alive but older than 24h is also treated as stale and transitioned to `failed` with `staleness: ttl-exceeded`. Pid-dead records become `failed` with `staleness: pid-dead`.
- **Override** is `pi-squire task --force`, which bypasses the active-write check. When used, the runtime writes `forced: true` and `forcedAt: <ISO-8601>` on the new record so `pi-squire status` and the audit trail surface the override. The skill explicitly forbids the agent from passing `--force`; only humans use it.
- **`pi-squire status` proactively reaps stale background records** on every call, so completed-but-not-marked jobs free their slot without requiring a separate sweep command.
- **Filesystem note:** `proper-lockfile` relies on `O_EXCL` create atomicity, which holds on local APFS/ext4. Home directories on NFS would need an advisory `flock(2)` fallback; v1 documents the limitation rather than implementing it.

### Optional future mode

- `task --worktree` can create an isolated worktree for large or risky changes, but this is not the default.

## Codex Plugin

Package as a Codex plugin with:

- `.codex-plugin/plugin.json`
- `skills/pi-delegation/SKILL.md`
- `scripts/pi-squire.mjs`

No `.mcp.json`, no MCP server. The skill is the only agent-facing surface; the CLI is the only runtime surface.

### Codex Skill

Provide a single skill named `pi-delegation`.

Skill description (drives implicit invocation):

> Use proactively when a task is mechanical, repetitive, benefits from an independent second opinion, or when Codex is stuck after one reasonable attempt. Do not wait for the user to ask for Pi.

Skill rules taught in `SKILL.md`:

- Codex is the editor-in-chief; Pi is auxiliary.
- Use `pi-squire brainstorm` for options and rubber-ducking.
- Use `pi-squire review` or `pi-squire challenge` before finalizing meaningful diffs.
- Use `pi-squire task` for scoped grunt work or simple implementation tasks.
- After `pi-squire task --bg` or any other call, remember the returned job id. To check on or fetch results from a specific delegation, use `--job <id>`.
- Review Pi output before integrating it into the final answer.

### Agent-facing CLI surface

The skill teaches the agent these invocations. Prompts are passed via heredoc, `--prompt-file`, or trailing argument:

```text
pi-squire setup
pi-squire brainstorm                  <prompt>
pi-squire review                      <prompt>
pi-squire challenge                   <prompt>
pi-squire task        [--bg]          <prompt>
pi-squire status      [--job <id>]
pi-squire result       --job <id>
pi-squire cancel       --job <id>
```

Every command supports `--json` for structured output the agent can parse. The runtime auto-generates the `acpx` session name and a stable job id per call; both are echoed in the output so the agent can keep them in scrollback. Status visibility is the primary observability surface for both the agent and the human — `pi-squire status` is the canonical way anyone sees what's running, what completed, and what failed.

**Long-running commands are foreground by default.** `task` (and to a lesser extent `review`/`challenge`) can take minutes; the runtime simply blocks until `acpx` returns. Both Codex CLI and Claude Code transparently handle multi-minute shell calls — Codex turns the process into a pollable session up to its `background_terminal_max_timeout` (5 min default), and Claude Code's Bash tool defaults to a 2-min timeout with a 10-min ceiling. The skill instructs the agent to invoke `pi-squire task` synchronously when the user is waiting for the answer in this turn. `--bg` exists for fire-and-forget cases where the user has explicitly handed off the work and will pick up the result in a later turn or out-of-band; the agent then surfaces the job id and uses `pi-squire status` / `result` on the next relevant turn.

### Codex Proactive Policy

Codex should proactively call Pi when:

- The task is straightforward but time-consuming.
- A second implementation pass could reveal a simpler path.
- The host is debugging and has not found the issue after one attempt.
- The user asks for review, hardening, cleanup, tests, docs, or migration work.
- The host is about to finalize a non-trivial diff and a cheap second opinion is useful.

Codex should not call Pi when:

- The request is tiny and faster to do directly.
- The task involves secrets, credentials, payment, auth, or destructive data changes.
- The user asked not to delegate.
- The workspace state is too conflicted to safely allow another writer.

## Proactive Delegation Policy

This policy applies to Codex.

Use Pi proactively for:

- mechanical edits across many files
- docs, tests, fixtures, examples, changelog drafts
- first-pass bug investigation
- cheap alternative plan generation
- second opinion on diffs
- adversarial review of design assumptions
- rubber-duck summaries when the host is stuck

Do not use Pi proactively for:

- one-line changes
- high-risk auth/security/payment/data-loss logic
- secrets or credential handling
- destructive shell/database operations
- situations where the user explicitly wants only the Primary Agent

Default escalation ladder:

1. Host frames the task.
2. If the task is simple but non-trivial, host delegates to Pi.
3. If the task is risky or ambiguous, host asks Pi for read-only challenge/review first.
4. If Pi writes code, host inspects the diff.
5. Host verifies with tests/checks where appropriate.
6. Host finalizes or rejects Pi's work.

## Prompt Contracts

Every Pi prompt should include:

- workspace goal
- exact task
- constraints
- whether writes are allowed
- what output is expected
- that Pi should avoid asking the human unless genuinely blocked

Write-capable prompt template:

```text
You are assisting the Primary Agent. Work in this repository checkout.

Task:
<task>

Constraints:
- Keep the change scoped.
- Prefer existing project patterns.
- Do not commit.
- Do not ask the human unless blocked.
- Run focused verification if practical.

Output:
- Summary of changes
- Files changed
- Verification performed or why it was not run
```

Read-only challenge template:

```text
You are reviewing work for the Primary Agent.

Task:
Challenge the current approach and identify risks, simpler alternatives, or missing verification.

Rules:
- Read-only. Do not edit files.
- Prioritize correctness, regressions, edge cases, and hidden assumptions.
- Be concise and specific.
```

## Permissions

One command, one permission posture. Map runtime commands to `acpx` flags:

```text
brainstorm           -> --deny-all
review, challenge    -> --approve-reads --non-interactive-permissions fail
task                 -> --approve-all
```

Notes:

- `--non-interactive-permissions fail` is set only on `review`/`challenge`. Since these are nominally read-only, any non-read approval request from Pi indicates the prompt or adapter is doing something unexpected; surfacing it as a hard error is more useful than silently denying.
- For `brainstorm` (`--deny-all`) and `task` (`--approve-all`), `acpx`'s default `--non-interactive-permissions deny` is acceptable: `--deny-all` rejects everything anyway, and `--approve-all` means no prompts ever fire.

This is intentionally coarse for v1. A later version can expose finer write policy controls if `acpx` and the Pi adapter support them cleanly.

## Background Execution

Foreground (default for every command):

- Block until `acpx` returns. The host harness (Codex CLI, Codex Desktop, Claude Code) is responsible for polling the running shell, not us.
- Stream final output, then exit.

Background (`task --bg` only):

- Spawn a detached worker, write the job id to stdout, exit immediately.
- The worker writes its log and result files, updates the job record, and dies.
- Used when the user has explicitly handed off the work and is not waiting in this turn.

Default:

- `brainstorm` / `review` / `challenge`: always foreground. They are short and read-only; backgrounding them buys nothing.
- `task`: foreground unless the agent (or human) passes `--bg`. The skill nudges `--bg` only when the user has signalled "do this and tell me later" or when the work is expected to exceed the harness's shell-timeout ceiling (5 min on Codex CLI, 10 min on Claude Code).

Rationale: agent harnesses already poll long-running shells natively, so no in-runtime `wait` primitive is required for MVP. If a future host has a stricter shell timeout we revisit by adding `pi-squire wait`.

## Result Handling

After `pi-squire task`:

- Host reads `pi-squire result`.
- Host checks `git status` and relevant diff.
- Host verifies changed files before final response.
- Host may ask Pi for a follow-up only after reviewing the first result.

The final user-facing answer is written by the Primary Agent, not Pi.

## Versioning and Pinning

Pin `acpx` for reproducibility.

Reason:

- `acpx` currently documents itself as alpha.
- CLI/runtime interfaces may change.

Recommended v1 approach:

- plugin package declares a tested `acpx` version range
- setup reports the detected version
- runtime refuses known-incompatible versions
- no direct imports from `acpx` internals in v1

## MVP Scope

MVP should include:

- `pi-squire` CLI runtime (`scripts/pi-squire.mjs`)
- Codex `pi-delegation` skill with proactive policy and CLI usage
- setup/status/result/cancel
- same-checkout write-capable task runs
- job tracking keyed by per-workspace job id (no host-session scoping)
- one named `acpx` Pi session per Pi job
- version checks

Defer:

- MCP server wrapper (only revisit if a host without a shell/skill mechanism needs it)
- Codex hooks (e.g. cleanup on session end)
- worktree mode
- deep NDJSON event visualization
- model selection
- Pi configuration management
- marketplace packaging polish
- automatic final-review gate enabled by default

## Open Questions

_None remaining._

## Resolved Decisions

- **Plugin surface:** Skill + CLI only. No MCP server. The skill describes when to delegate; the agent runs `pi-squire` directly. (Reason: Codex skills already drive proactive use, the runtime is naturally CLI-shaped, and humans can run the same commands the agent runs.)
- **Host session identity:** Not tracked. The runtime does not know or care which Codex conversation created a Pi Job. `pi-squire status` lists all jobs in the workspace; agents and humans alike find specific jobs by the job id printed when the job was created. (Reason: the concept was load-bearing only when `resume-candidate` and per-conversation `status` filtering existed; with Pi disposable per subtask, neither is needed. Dropping it removes agent-managed UUIDs, the `--session` flag everywhere, and a field from every job record.)
- **Pi Job semantics:** A Pi Job is one delegated subtask, owning one named `acpx` session. Pi is disposable per subtask: there is no `resume-candidate`, no cross-delegation continuity, and the runtime auto-generates the `acpx` session name. The Primary Agent is the only place where context accumulates across calls. (Reason: simplifies the model dramatically — agent doesn't track Pi session names, and the "is this resumable?" branch goes away.)
- **Codex-plugin-cc reuse:** Lift the central-dispatcher script, `runTrackedJob()` lifecycle, `state.mjs` persistence boundary, thin forwarding subagent pattern, and adversarial-review prompt scaffold. Skip the app-server broker (`acpx` already provides persistent named sessions). Defer hooks. See `codex-plugin-cc-reference.md` for the full pattern-by-pattern reuse table.
- **Long-running shell commands:** Default to foreground/blocking. Codex CLI converts long shells into pollable sessions up to a 5-minute `background_terminal_max_timeout`; Claude Code's Bash tool defaults to a 2-minute timeout with a 10-minute ceiling and supports `run_in_background`. No in-runtime `wait` primitive in MVP. (Reason: agent harnesses already poll long shells natively; building our own would duplicate the harness work.)
- **Concurrency / locking:** Workspace key is canonical `git rev-parse --show-toplevel` (cwd fallback). The job-state JSON is the lock; all mutations happen under `proper-lockfile` with a 5-second staleness ceiling. Stale-job recovery uses pid-liveness + 24h hard TTL. Scope is per-workspace across all host sessions. Override is `pi-squire task --force` with audit fields (`forced`, `forcedAt`); the skill forbids the agent from passing `--force`.
- **Permissions:** One command, one posture. `brainstorm → --deny-all`; `review`/`challenge → --approve-reads --non-interactive-permissions fail`; `task → --approve-all`. No `--read` flag on `brainstorm`.
- **State location:** Centralized under `~/.pi-squire/state/<workspace-hash>/`, not inside the workspace. `<workspace-hash> = sha256(canonical git toplevel || cwd)[:16]`. State lives outside `~/.codex/plugins/pi-squire/` (the Codex-managed install dir) so plugin updates can't wipe it. (Reason: avoids polluting the repo, supports read-only checkouts, and survives plugin reinstall.)
- **Names:** Plugin manifest name is `pi-squire`; slash commands are `/pi-squire:<verb>`; CLI binary is `pi-squire`; skill is `pi-delegation`. (Reason: `pi` alone collides with the actual Pi coding agent's CLI; `pi-squire` is unambiguous and matches the repo identity.)
- **`--write` flag for humans:** Not added. The command name (`task` vs `review`/`challenge`/`brainstorm`) carries the write/non-write signal; the per-workspace write-lock, mandatory diff inspection, and audit fields are the actual safety. A `--write` flag would only add friction without new failure-mode coverage.
