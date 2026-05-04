# Codex Pi Delegation

Codex Pi Delegation lets a Codex session delegate scoped work to Pi while Codex remains responsible for integration and the final answer.

## Language

**Primary Agent**:
The Codex agent that owns the user-facing work and final decision.
_Avoid_: host agent, main agent

**Host Session**:
A single Codex Desktop thread or Codex CLI conversation. Conceptually distinct from any other Codex window, but the runtime does not identify or scope by Host Session — `pi-squire status` returns all Pi Jobs for the workspace, and the agent picks out its own jobs by the job ids in its scrollback. The concept exists to describe *who* is delegating; it does not appear in the runtime's state.
_Avoid_: workspace session, terminal session

**Pi Job**:
One delegated subtask owned by a Host Session. A Pi Job maps one-to-one to a named `acpx` Pi session. Pi is treated as a disposable per-subtask helper: Pi Jobs are not reused across delegations, and there is no resume facility. The runtime auto-generates the `acpx` session name; the Primary Agent never picks or remembers it.
_Avoid_: task session, worker, "one Pi run", "long-lived Pi session"

**Pi Turn**:
A single prompt-and-response exchange inside a Pi Job. A Pi Job typically has one Pi Turn but can naturally have a few when Pi asks a clarifying question and continues. New delegations start fresh Pi Jobs; they do not add Turns to an old one.
_Avoid_: run, task, exchange

**Codex Plugin**:
The Codex-facing package (manifest name `pi-squire`) that ships a Codex skill and a `pi-squire` CLI for delegation. It does not ship an MCP server; the agent invokes the CLI directly under guidance from the skill.
_Avoid_: Claude plugin, host adapter, MCP server bundle, "pi-companion" (legacy working name)

## Relationships

- A **Host Session** contains zero or more **Pi Jobs**.
- A **Pi Job** maps one-to-one to a named `acpx` Pi session.
- A **Pi Job** consists of one or more **Pi Turns** against that named session.
- A **Primary Agent** may create many **Pi Jobs** — one per delegated subtask — and remains responsible for reviewing Pi output after every **Pi Turn**.
- Pi Jobs are not reused across subtasks. Continuity across delegations lives in the **Primary Agent**, not in Pi.
- A **Codex Plugin** connects the **Primary Agent** to Pi through `acpx`.

## Example Dialogue

> **Dev:** "Can this **Host Session** ask Pi to continue the previous delegated work?"
> **Domain expert:** "No — each delegation is a fresh **Pi Job**. Pi is disposable per subtask; if continuity is needed, the **Primary Agent** carries it by giving the new Pi Job whatever prior context it needs."

## Flagged Ambiguities

- "session" was used as if a Codex conversation owned one long-lived Pi conversation; resolved: a **Pi Job** owns one named `acpx` Pi session.
- "Pi Job" was previously described as a single delegated _run_, then revised to a multi-turn conversation; resolved (final): a **Pi Job** is one delegated subtask with one named `acpx` session and is not reused across subtasks. Pi is disposable per subtask. **Pi Turns** still exist as the prompt/response unit inside one job.
