# Agent Orchestrator Design

## Goal

Build a lightweight system for running long-lived coding agents on a home Linux server, keeping them isolated, interactive, remotely steerable, and eventually controllable from a phone.

The core rule:

> **Do not build the spaceship first.**
>
> Each layer should be useful on its own and should only be added after the previous layer proves valuable in real use.

---

## Core Architecture

```text
                           Phone / Telegram / Matrix
                                      |
                                      v
                              +----------------+
                              |     Hermes     |
                              |    "brain"     |
                              +----------------+
                                      |
                               MCP / narrow RPC
                                      |
                                      v
+----------------------------------------------------------------+
|                         Linux host                              |
|                                                                |
|   +----------------------+                                     |
|   |   Bridge container   |                                     |
|   |                      |                                     |
|   | - sandbox lifecycle  |                                     |
|   | - tmux control       |                                     |
|   | - status/snapshots   |                                     |
|   | - event emission     |                                     |
|   +----------+-----------+                                     |
|              |                                                 |
|              | controls                                        |
|              v                                                 |
|      +-------------------+       +-------------------+          |
|      | Sandbox / microVM |       | Sandbox / microVM |   ...    |
|      |                   |       |                   |          |
|      | tmux              |       | tmux              |          |
|      | coding agent CLI  |       | coding agent CLI  |          |
|      | project workspace |       | project workspace |          |
|      | Docker            |       | Docker            |          |
|      +-------------------+       +-------------------+          |
|                                                                |
+----------------------------------------------------------------+
```

The current preferred runtime is **Docker Sandboxes / KVM-backed microVMs on Linux**.

The sandbox is the security boundary. Inside it, the coding agent may have normal/rootful Docker access because destroying the sandbox should be acceptable.

---

# Component Responsibilities

## 1. Sandbox

A sandbox contains:

- project checkout/worktree
- coding agent CLI
- `tmux`
- Docker / Docker Compose
- normal development tooling
- optional notification helper script

The agent is allowed to:

- edit project files
- build containers
- run services
- run tests
- destroy its own Docker environment
- execute arbitrary commands inside the sandbox

The agent must **not** have direct access to:

- host Docker socket
- host filesystem outside explicitly shared project data
- host secrets
- unrelated projects
- `/dev/kvm`
- infrastructure control

The VM/microVM is the actual isolation layer.

---

## 2. tmux

`tmux` is the universal interactive adapter around coding agents.

Each live agent runs inside a named tmux session/window.

Benefits:

- SSH disconnects do not kill the agent
- operator can attach directly from a laptop
- bridge can capture terminal output
- bridge can inject input
- bridge can detect attached clients
- agent implementation remains irrelevant

The system should not care whether the worker is:

- Codex
- Claude Code
- OpenCode
- another CLI agent

If it runs in a terminal, tmux can wrap it.

---

## 3. Bridge

The bridge is **infrastructure, not intelligence**.

It runs in its own container on the Linux host.

It is the only component with enough host capability to control the sandbox runtime.

Conceptually it owns:

```text
sandbox lifecycle
tmux lifecycle
sandbox metadata
terminal snapshots
input injection
basic events
```

It exposes a deliberately narrow API.

Possible operations:

```text
list_agents()
start_agent(project, task?)
stop_agent(agent_id)
get_status(agent_id)
get_terminal_snapshot(agent_id)
send_input(agent_id, text)
interrupt(agent_id)
```

Avoid a generic:

```text
exec_arbitrary_host_command(...)
```

because that collapses the entire security boundary.

The bridge may have `/dev/kvm` or whatever host capability the sandbox runtime requires.

Hermes does not.

---

## 4. Hermes

Hermes is the high-level "brain".

Responsibilities:

- Telegram / Matrix interaction
- understanding user intent
- deciding which sandbox/project the user means
- summarizing agent state
- deciding whether to notify the user
- invoking bridge operations
- optionally steering worker agents

Hermes should **not** know:

- raw KVM details
- `sbx` implementation details
- tmux naming conventions
- host process management

Hermes sees only the bridge API.

This keeps Hermes replaceable.

---

# Control Flows

## Laptop / direct interactive mode

```text
MacBook
   |
   | SSH
   v
Linux server
   |
   v
tmux attach
   |
   v
coding agent inside sandbox
```

This path should stay simple.

Do **not** proxy SSH through the bridge just to maintain architectural purity.

When the human is directly attached, Hermes should generally back off.

---

## Phone / Hermes mode

```text
Phone
  |
Telegram / Matrix
  |
  v
Hermes
  |
  v
Bridge
  |
  v
sandbox + tmux + worker agent
```

Example:

```text
"Fix the retry bug in `project XYZ`"
```

Hermes:

1. identifies project XYZ
2. asks bridge for current state
3. starts or reuses an agent
4. sends the task
5. later receives completion/progress information
6. summarizes it back to the user

---

# Interactive vs Autonomous Mode

A useful distinction:

```text
AUTONOMOUS
INTERACTIVE
```

### Autonomous

Hermes may:

- observe
- inject commands
- start work
- notify user

### Interactive

Human is currently attached over SSH/tmux.

Hermes should:

- avoid steering
- avoid injecting input
- optionally suppress noisy notifications
- still allow passive observation

tmux can expose whether clients are attached.

That is enough for MVP.

Do not try to perfectly attribute every keystroke.

---

# Notifications

There are two possible notification mechanisms.

## Early/simple version

Bake a script into every sandbox:

```bash
notify-agent "finished: implemented retry handling"
```

Shared agent instructions say:

```text
When you finish a task or become blocked, call notify-agent.
```

This is **not** an MCP tool.

It is simply a shell script plus instructions injected into the agent context.

The script calls a narrow endpoint exposed by infrastructure.

Advantages:

- extremely simple
- agent-controlled semantic completion
- works regardless of which coding agent is used

---

## Later version

The bridge emits structured events:

```text
agent.started
agent.finished
agent.failed
agent.blocked
sandbox.created
sandbox.destroyed
```

Hermes subscribes and decides whether the event deserves a user notification.

Important distinction:

> Bridge emits facts. Hermes decides meaning.

A sandbox should not need Telegram credentials.

---

# Important Edge Cases

## 1. Human and Hermes controlling the same agent

Avoid simultaneous steering.

Simple solution:

- if tmux has an attached human client, mark session `INTERACTIVE`
- Hermes becomes read-only / hands-off

Do not build complicated command ownership tracking initially.

---

## 2. Context drift

Hermes may see only:

- snapshots
- recent terminal state
- task metadata

while the coding agent saw the whole conversation.

Hermes should therefore avoid pretending it fully understands the worker's reasoning.

Useful bridge metadata:

```text
agent_id
project
started_at
current_task
status
last_activity
tmux_session
human_attached
```

---

## 3. Bridge restart

The bridge should not require perfect in-memory state.

On startup it should be able to reconstruct reality by querying:

- existing sandboxes
- tmux sessions
- metadata files / labels

The infrastructure state should be discoverable.

---

## 4. Duplicate requests

Hermes or messaging systems may retry.

Operations such as:

```text
start_agent(...)
stop_agent(...)
```

should eventually become idempotent.

Not necessary for the first prototype, but important once phone control is real.

---

## 5. Sandbox deletion and project state

If using cloned/disposable filesystems, deleting the sandbox may delete unpushed work.

Before destruction, eventually require one of:

- git commit
- git push
- exported patch
- explicit "discard"

Do not solve this before the workflow proves useful.

---

# Implementation Roadmap

The roadmap is intentionally staged.

---

## V0 — Manual Playtest

**Goal:** prove that the workflow itself feels useful.

No custom bridge.
No Hermes integration.
No event system.

Use:

```text
Linux server
Docker Sandboxes
tmux
coding agent CLI
SSH
```

Workflow:

```text
ssh server
tmux new-window
start sandbox
start coding agent
detach
come back later
attach
```

Test:

- Is sandbox startup fast enough?
- Does Docker-inside-sandbox work comfortably?
- Does tmux interaction feel natural?
- Are project files easy to preserve?
- Do long-running agents actually save time?
- Do multiple concurrent agents feel useful or just noisy?

### Success condition

You naturally use this setup repeatedly without forcing yourself.

If V0 is annoying, stop here and fix the workflow before building infrastructure.

---

## V0.5 — Notification Script

Add only:

```text
notify-agent
```

Each sandbox gets the script automatically.

Agent instructions include:

```text
When finished or blocked, call notify-agent with a short summary.
```

Initially the script can send directly to:

- Telegram
- Matrix
- a tiny host HTTP receiver

No Hermes required.

### Goal

Answer one question:

> Is "agent works while I leave, then pings me" genuinely useful?

---

## V1 — Bridge

Only after V0/V0.5 prove useful.

Build the narrow bridge.

Responsibilities:

```text
list sandboxes
create sandbox
destroy sandbox
inspect status
capture tmux pane
send tmux input
interrupt agent
```

Bridge runs in its own container.

It is the only component allowed to touch sandbox/KVM infrastructure.

### Do not add yet

- event bus
- database
- complex scheduler
- authentication hierarchy
- full audit logging
- generic exec API
- mobile UI

---

## V1.5 — Persistent Metadata + Events

Add stable identities:

```text
agent-42
agent-43
```

Track:

```text
agent id
project
task
sandbox id
tmux session
status
human attached
timestamps
```

Add basic events.

Example:

```json
{
  "type": "agent.finished",
  "agent_id": "agent-42",
  "project": "tiktok-farm"
}
```

No Kafka.
No NATS unless there is a real reason.

A tiny internal stream/websocket/HTTP callback is enough.

---

## V2 — Hermes as Observer

Connect Hermes to the bridge through MCP or another narrow interface.

Initially Hermes should mostly:

- list agents
- inspect state
- summarize terminal snapshots
- answer "what is happening?"
- notify about finished/blocked work

Do **not** let Hermes aggressively steer agents yet.

This is the safest way to test whether the "brain" abstraction is actually valuable.

---

## V3 — Phone Steering

Once observation works well, allow Hermes to:

```text
start_agent
send_input
interrupt
resume
stop
```

Now the phone becomes a lightweight operations console.

Typical interaction:

```text
User:
"What is the TikTok project doing?"

Hermes:
"The consumer refactor finished tests but one integration test fails."

User:
"Tell it to fix that."

Hermes:
send_input(agent-42, "...")

...
Hermes:
"Done. Tests are green."
```

This is the first version that delivers the original dream:

> Keep projects moving while away from the laptop.

---

## V4 — Human Presence / Mode Awareness

Add explicit:

```text
human_attached = true/false
```

tmux hooks or inspection can update it.

When human is attached:

```text
Hermes -> observe only
```

When human detaches:

```text
Hermes -> autonomous control allowed
```

Only add idle-time heuristics if this becomes annoying.

---

## V5 — Reliability Polish

Only after sustained usage.

Possible additions:

- idempotent API calls
- automatic crash recovery
- stale sandbox cleanup
- durable event history
- project templates
- resource quotas
- concurrency limits
- network policy
- sandbox TTL
- structured logs
- health checks
- Git safety checks before destruction

These are operational improvements, not core product features.

---

# Explicit Non-Goals

Do not accidentally build these early:

### A custom SSH proxy

Direct SSH + tmux is already good.

### A custom terminal protocol

tmux already provides one.

### A custom sandbox implementation

Use Docker Sandboxes/KVM first.

### A generalized orchestration platform

You are building a personal development tool, not Kubernetes.

### Perfect activity attribution

Knowing whether the human is attached is probably enough.

### A massive event-driven architecture

Start with callbacks or simple polling.

### A custom coding agent

Use whichever CLI agent is best.

---

# Security Model

The main protection comes from the sandbox VM boundary.

```text
untrusted:
    worker coding agent
    code generated by worker
    project containers

semi-trusted:
    Hermes

trusted infrastructure:
    bridge

host:
    final trust boundary
```

Important properties:

- worker never sees host Docker
- worker never sees `/dev/kvm`
- Hermes never sees `/dev/kvm`
- Hermes cannot run arbitrary host commands
- only bridge can create/destroy sandboxes
- bridge exposes only narrow high-level operations

If the worker gets root **inside its microVM**, that is acceptable by design.

---

# Guiding Principle

Every iteration must answer a concrete usability question.

```text
V0:
"Do I enjoy using remote sandboxed agents?"

V0.5:
"Are completion notifications useful?"

V1:
"Do I need programmatic lifecycle control?"

V2:
"Does Hermes add value by understanding and summarizing state?"

V3:
"Is phone-based steering actually useful?"

V4+:
"What operational pain repeatedly shows up?"
```

If an iteration does not solve an observed problem, don't build it.

---

# Minimal First Experiment

The immediate test should be boring:

1. Install Docker Sandboxes on the Linux server.
2. Pick one non-critical project.
3. Start one sandbox.
4. Run the coding agent in tmux.
5. Give it a real task.
6. Disconnect SSH.
7. Come back later.
8. Reattach.
9. Check whether the experience felt useful.

Then run two agents simultaneously.

Only after that decide whether the bridge deserves to exist.

---

## Desired End State

Eventually:

```text
Phone:
"What's running?"

Hermes:
"Three agents:
 - tape-robot: implementing rail geometry
 - tiktok-farm: fixing queue consumer restart
 - project XYZ: tests complete"

Phone:
"Tell tiktok-farm to also add the regression test."

Hermes:
"Done."

...later...

Hermes:
"tiktok-farm finished. Regression test added and suite passes."
```

Meanwhile, from a laptop:

```bash
ssh home-server
tmux attach
```

and you can directly take over any session.

That is the target.

Everything between V0 and that state should justify its own existence.
