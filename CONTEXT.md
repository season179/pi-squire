# Codex Pi Delegation

Codex Pi Delegation lets a Codex session delegate scoped work to Pi while Codex remains responsible for integration and the final answer.

## Language

**Primary Agent**:
The Codex agent that owns the user-facing work and final decision.
_Avoid_: host agent, main agent

**Host Session**:
A single Codex Desktop thread or Codex CLI conversation that keeps track of its delegated Pi work.
_Avoid_: workspace session, terminal session

**Pi Job**:
One delegated run sent from Codex to Pi; it owns one named `acpx` Pi session.
_Avoid_: task session, worker

**Codex Plugin**:
The Codex-facing package that exposes skills and tools for delegation.
_Avoid_: Claude plugin, host adapter

## Relationships

- A **Host Session** contains zero or more **Pi Jobs**.
- A **Pi Job** maps one-to-one to a named `acpx` Pi session.
- A **Primary Agent** may create many **Pi Jobs**, but remains responsible for reviewing Pi output.
- A **Codex Plugin** connects the **Primary Agent** to Pi through `acpx`.

## Example Dialogue

> **Dev:** "Can this **Host Session** ask Pi to continue the previous delegated work?"
> **Domain expert:** "Yes, by resuming the specific **Pi Job**; the job's named `acpx` session is the Pi conversation."

## Flagged Ambiguities

- "session" was used as if a Codex conversation owned one long-lived Pi conversation; resolved: a **Pi Job** owns one named `acpx` Pi session.
