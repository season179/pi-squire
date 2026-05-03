# Codex Pi Delegation Plugin Spec

Status: draft

## Goal

Build a Codex plugin that lets Codex delegate work to the Pi Coding Agent through `acpx`.

The primary agent can be either a Codex Desktop session or a Codex CLI session.

The primary agent remains the editor-in-chief. Pi is a cheaper auxiliary agent used for grunt work, independent diagnosis, brainstorming, rubber-ducking, and second-opinion review.

The plugin should encourage proactive use of Pi. The human should not need to say "ask Pi" for the host agent to use it.

## Non-Goals

- Do not implement model selection in this plugin. Pi uses whatever model/provider the user has configured in Pi.
- Do not require a separate git worktree by default.
- Do not replace Codex as the final reviewer/integrator.
- Do not make Pi a native Codex subagent runtime. It is reached through plugin commands/tools backed by `acpx`.

## Architecture

Use a Codex plugin plus a small runtime.

```text
Codex Desktop session
        |
        v
Codex plugin
  skills + MCP tools
        |
        v
pi-companion runtime
        |
        v
acpx CLI
        |
        v
Pi ACP adapter / Pi Coding Agent

Codex CLI session
        |
        v
Codex plugin/tooling entrypoint
        |
        v
pi-companion runtime
        |
        v
acpx CLI
        |
        v
Pi ACP adapter / Pi Coding Agent
```

`acpx` is the transport/runtime boundary. It already provides ACP client behavior, Pi adapter resolution, persistent sessions, named sessions, prompt queueing, cancel, status, permission modes, cwd scoping, and JSON/quiet output.

## Runtime

Provide a Node CLI:

```bash
node scripts/pi-companion.mjs <command> [options] [prompt]
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
resume-candidate
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

Purpose: cheap second brain without workspace mutation.

Behavior:

- Prefer `acpx --deny-all pi exec <prompt>` when the host supplies enough context directly.
- Use `--approve-reads` only if the host explicitly wants Pi to inspect local files.
- Output should be concise options, risks, or questions.

### `task`

Purpose: write-capable delegated work.

Behavior:

- Create or resume the persistent named `acpx` session for this job:

```bash
acpx --cwd <workspace> --approve-all pi -s <job-session> <prompt>
```

- The host adapter decides foreground vs background.
- The runtime records job state before launching.
- The runtime captures stdout/stderr or NDJSON event logs.
- The runtime snapshots git status before and after the run.

Default posture:

- `task` is write-capable.
- Use the same checkout by default.
- Do not require a worktree.
- Allow optional future `--worktree` mode for invasive or long-running work.

### `status`

Shows active and recent Pi jobs for the current workspace.

Should include:

- job id
- command kind
- status: queued, running, completed, failed, canceled
- foreground/background
- workspace root
- job session name
- host session id when available
- pid if available
- started/completed timestamps
- latest progress lines

### `result`

Shows final stored output for a job.

Should include:

- final Pi output
- error output if failed
- `acpx` session identifiers when available
- touched/possibly touched files when known
- reminder that host agent must review before finalizing

### `cancel`

Cancels an active Pi job:

```bash
acpx --cwd <workspace> pi cancel -s <job-session>
```

If the runtime spawned a detached worker, also clean up local job state.

### `resume-candidate`

Finds the latest resumable Pi task for this workspace and host session.

Used by Codex tools to decide whether to continue prior Pi work.

## Session Model

Each Pi job is one named Pi session.

- A Codex Desktop thread and a Codex CLI conversation are different host sessions, even when they are opened on the same workspace.
- A host session can have many Pi jobs.
- The runtime derives a stable job session name from host kind, host session id, workspace root, and job id.
- A Pi job maps one-to-one to a named `acpx` Pi session.
- Resuming a Pi job means sending another prompt to that job's named `acpx` session.
- Starting a new Pi job means starting a fresh named `acpx` Pi session.
- The host session id is only used to group and find jobs for the current Codex conversation; it is not itself the Pi conversation.
- If the Codex host cannot provide a stable host session id, the runtime must report that limitation because `status` and `resume-candidate` cannot reliably scope jobs to the current Codex conversation.

## Job State

Store plugin state under the host plugin data directory when available, otherwise under a temp fallback.

Each job record:

```json
{
  "id": "pi-task-...",
  "host": "codex-desktop|codex-cli",
  "hostSessionId": "codex-session-...",
  "kind": "review|challenge|brainstorm|task",
  "workspaceRoot": "/abs/path",
  "cwd": "/abs/path",
  "jobSessionName": "pi-task-...",
  "status": "queued|running|completed|failed|canceled",
  "pid": 12345,
  "promptPreview": "short text",
  "write": true,
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

Keep the newest 50 jobs per workspace by default.

## Same-Checkout Safety Model

Same checkout is the default because this should feel ergonomic, direct, and integrated into the current Codex workflow.

Safety comes from mode selection, job tracking, and host review, not mandatory checkout isolation.

Rules:

- Allow multiple read-only Pi jobs in parallel.
- Allow only one Pi write job per workspace at a time.
- Do not start a second Pi write job while one is active unless the human explicitly overrides.
- The host agent should avoid editing the same files while a Pi write job is active.
- After a write-capable Pi job, the host must inspect the diff before claiming completion.
- The host must run or request appropriate verification before finalizing.

Optional future mode:

- `task --worktree` can create an isolated worktree for large or risky changes, but this is not the default.

## Codex Plugin

Package as a Codex plugin with:

- `.codex-plugin/plugin.json`
- `skills/`
- `.mcp.json`
- MCP server script
- `scripts/pi-companion.mjs`

### Codex Skills

Provide a skill such as `pi-delegation`.

Skill trigger/description should say:

> Use proactively when a task is mechanical, repetitive, benefits from an independent second opinion, or when Codex is stuck after one reasonable attempt. Do not wait for the user to ask for Pi.

Skill rules:

- Codex is the editor-in-chief.
- Use Pi for cheap parallel cognition and delegated execution.
- Use `pi_brainstorm` for options and rubber-ducking.
- Use `pi_review` or `pi_challenge` before finalizing meaningful diffs.
- Use `pi_task` for scoped grunt work or simple implementation tasks.
- Review Pi output before integrating it into the final answer.

### Codex MCP Tools

Expose:

```text
pi_setup
pi_brainstorm
pi_review
pi_challenge
pi_task
pi_status
pi_result
pi_cancel
```

Tool descriptions should contain proactive usage guidance, not just neutral API descriptions.

Example `pi_task` description:

> Delegate a scoped coding task to Pi through acpx. Use proactively for mechanical edits, simple bug fixes, docs/tests, or a cheaper first implementation pass. The host agent remains responsible for reviewing the diff and final answer.

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
- situations where the user explicitly wants only the host agent

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
You are assisting the primary agent. Work in this repository checkout.

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
You are reviewing work for the primary agent.

Task:
Challenge the current approach and identify risks, simpler alternatives, or missing verification.

Rules:
- Read-only. Do not edit files.
- Prioritize correctness, regressions, edge cases, and hidden assumptions.
- Be concise and specific.
```

## Permissions

Map modes to `acpx` permissions:

```text
brainstorm, no file inspection -> --deny-all
review/challenge/diagnosis     -> --approve-reads --non-interactive-permissions fail
task/rescue with writes         -> --approve-all
```

This is intentionally coarse for v1. A later version can expose finer write policy controls if `acpx` and the Pi adapter support them cleanly.

## Background Execution

Foreground:

- Use for tiny tasks and quick brainstorming.
- Return Pi output directly.

Background:

- Use for long reviews, broad mechanical edits, test generation, or open-ended investigation.
- Return a job id immediately.
- User or host can call status/result/cancel.

Default:

- Review/challenge: ask or infer based on size.
- Task/rescue: background for open-ended or multi-step tasks; foreground for small scoped tasks.

## Result Handling

After `pi_task`:

- Host reads `pi_result`.
- Host checks `git status` and relevant diff.
- Host verifies changed files before final response.
- Host may ask Pi for a follow-up only after reviewing the first result.

The final user-facing answer is written by the host agent, not Pi.

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

- shared `pi-companion.mjs`
- Codex MCP tools
- Codex proactive delegation skill
- setup/status/result/cancel
- same-checkout write-capable task runs
- job tracking
- one named Pi session per Pi job
- version checks

Defer:

- worktree mode
- deep NDJSON event visualization
- model selection
- Pi configuration management
- marketplace packaging polish
- automatic final-review gate enabled by default

## Open Questions

- Should the project be named `pi-companion`, `pi-delegation`, or `pi-bridge`?
- Should write-capable Pi tasks require an explicit `--write` in human commands while proactive host usage can choose write internally?
- Should Codex expose both MCP tools and slash-command-like skills, or only MCP tools plus one delegation skill?
- How much of `codex-plugin-cc`'s job-state implementation should be copied versus rewritten as a smaller shared runtime?
- How exactly should Codex Desktop and Codex CLI expose stable host session ids to the runtime?
- Should `resume-candidate` only return unfinished/interrupted jobs, or also recently completed jobs that can accept follow-up prompts?
