# pi-agents-talk-to-each-other

A small [Pi](https://github.com/earendil-works/pi) package that lets multiple local Pi agents join the same room, see each other's status, and send messages/tasks to each other.

The first version is intentionally simple: it uses a local file-based room bus under `~/.pi/agent/rooms/`, so no daemon, sockets, database, or network server is required.

## Features

- Local rooms shared by all Pi processes on the same machine.
- Slash command for humans: `/room ...`.
- LLM-callable tools:
  - `room_whoami`
  - `room_list_agents`
  - `room_send_message`
  - `room_control_agent` (requires control permission)
  - `room_read_agent_history` (requires control permission)
  - `room_summarize_agent` (requires control permission)
- **Permission model**: by default, `room_control_agent` is disabled for all agents. Enable it per-agent with `/room control on`.
- Heartbeat/status for each agent:
  - agent id
  - idle/busy/tool/offline
  - current tool name and masked preview
  - cwd
  - model
  - current context usage when available
  - rough token totals when available
  - control flag `[control]`
- Follow-up delivery: if the target agent is busy, the received message is queued as a follow-up user message.
- Basic session control messages: `compact`, `reload`, `new_session` (only from agents with control enabled).

## Installation

From GitHub:

```bash
pi install git:github.com/Aprilarina/pi-agents-talk-to-each-other
```

For local development from this checkout:

```bash
pi install /absolute/path/to/pi-agents-talk-to-each-other
# or one-off:
pi -e /absolute/path/to/pi-agents-talk-to-each-other
```

After installing, restart Pi or run `/reload` in existing Pi sessions.

## Quick start

Open two or more Pi sessions and connect them to the same room:

```text
/room connect dev
```

Or open the interactive room browser:

```text
/room
```

`/room` without arguments opens a TUI browser with room list, agent counts, and actions.

`/room connect` only connects the current Pi process. New Pi sessions do not auto-connect by default.

If you explicitly want future Pi sessions to auto-connect, use either:

```text
/room connect dev --default
# or
/room default dev
```

List agents:

```text
/room list
```

Enable control for the current agent (allows `room_control_agent`):

```text
/room control on
```

Show your own id/status:

```text
/room whoami
```

Send a manual message:

```text
/room send agent-myhost-12345 Please inspect the failing tests. Reply to my agent id when done.
```

Leave the current room:

```text
/room leave
```

## Slash commands

:```text
/room                         Open room browser (TUI)
/room connect <room> [--default]
/room create <room> [--default]
/room leave [--keep-default]
/room control on|off
/room list [--stale]
/room send <agent-id> <message>
/room whoami
/room default <room|off>
/room status
```

## Control permission

By default, `room_control_agent` is **disabled** for all agents. This means:

- Agents can always use `room_whoami`, `room_list_agents`, and `room_send_message`.
- Agents **cannot** use `room_control_agent` (compact/reload/new_session on others) unless explicitly enabled.

To enable control for the current agent:

```text
/room control on
```

To disable:

```text
/room control off
```

The control flag persists across `/reload` (same process). It does **not** persist across full restarts (new process = new agent id), so you must re-enable it after restarting Pi.

Agents with control enabled are shown with `[control]` in `room_list_agents` output:

```text
agent-macbook-41841 idle [control] ctx:31% model:openai/gpt-5 cwd:/repo
agent-macbook-42109 tool tool:bash ctx:55% model:anthropic/claude cwd:/repo
```

## Tools available to the model

### `room_whoami`

Returns this agent's room identity and debug info.

Use it when an agent needs to include its own id in a delegated task or reply instruction.

### `room_list_agents`

Lists active agents in the current room.

Example output:

```text
agent-macbook-41841 idle [control] ctx:31% model:openai/gpt-5 cwd:/repo
agent-macbook-42109 tool tool:bash ctx:55% model:anthropic/claude cwd:/repo {"command":"npm test"}
```

### `room_send_message`

Sends a prompt-like user message to another agent by id.

Parameters:
- `to`: target agent id
- `message`: text to deliver
- `delivery`: `"steer"` (interrupt target's current turn) or `"followUp"` (wait for target to finish, default)

Use `delivery: "steer"` to urgently interrupt a busy agent that is doing something wrong. Use `delivery: "followUp"` (default) for normal messages that can wait.

Recommended delegation pattern:

> Do X. My agent id is `agent-...`. When done or blocked, call `room_send_message` back to me with the result.

### `room_control_agent`

Queues one of these actions on another agent:

- `compact` (works)
- `abort` (works — immediately cancels the target agent's current turn, like pressing Escape)
- `reload` (**DEPRECATED** — does not work, pi does not process slash commands from extension-injected messages; ask the user to run `/reload` manually)
- `new_session` (**DEPRECATED** — same reason; ask the user to run `/new` manually)

**Requires control permission.** If the agent does not have control enabled, the tool returns a permission error. Ask the user to run `/room control on`.

Use `abort` when a subordinate agent is doing something wrong and you need to stop it immediately. Use `compact` to reduce context. For `reload` and `new_session`, ask the user to run them manually.

### `room_read_agent_history`

Reads the last (or first) N lines of another agent's session transcript.

Parameters:
- `to`: target agent id
- `lines`: number of lines to return (default 50, max 500)
- `mode`: `"tail"` (last N, default) or `"head"` (first N)

**Requires control permission.** Use this to inspect what a subordinate agent has been doing before deciding to intervene.

### `room_summarize_agent`

Reads the last N turns of another agent's session, sends them through a model with a custom system prompt, and returns the summary.

Parameters:
- `to`: target agent id
- `turns`: number of last turns to include (default 10, max 50). A turn = one user message + assistant response + tool results.
- `systemPrompt`: instructions for the summarization model (e.g., "Summarize what files were changed and any errors encountered")
- `model`: model to use for summarization (optional, defaults to target agent's model)

**Requires control permission.** Use this to get a compressed view of a subordinate agent's activity without reading raw history lines.

The summarization runs as a separate `pi` subprocess in print mode with the specified model and system prompt. The transcript is built from the target agent's session file and passed as the task prompt.

## Storage layout

Runtime state is stored locally:

```text
~/.pi/agent/rooms/
  config.json
  control/
    <agent-id>.json          ← control permission flag
  <room>/
    agents/
      <agent-id>.json
    inbox/
      <agent-id>/
        <message-id>.json
    events.jsonl
```

Message files are deleted after delivery. `agents/*.json`, `control/*.json`, and `events.jsonl` are debug/runtime files and can be deleted if needed.

## Safety notes

- This is local-machine coordination only; it does not authenticate senders.
- Any Pi process that can write to `~/.pi/agent/rooms/<room>` can send messages.
- `room_control_agent` is gated by a per-agent control flag, but the flag file is not cryptographically protected.
- Tool argument previews are masked/truncated, but do not treat room debug files as a secure audit log.
- The extension never reads `.env` files or credentials by itself.

## Optional: cmux integration skill

If you run your agents in [cmux](https://github.com/manaflow-ai/cmux) (native macOS terminal for parallel AI agents), you can install an optional skill that lets the controlling agent reload a worker agent or create a new session for it via terminal automation:

```bash
npx skills add Aprilarina/pi-agents-talk-to-each-other -g -y
```

This installs the `cmux-room-reload` skill, which provides a workflow for:

1. Asking the worker agent for its `CMUX_SURFACE_ID` via `room_send_message`
2. Sending `/reload` or `/new` as keyboard input through `cmux send` (bypasses the `sendUserMessage` slash-command limitation)
3. Reconnecting the worker agent to the room with `/room connect <room>`

This skill is **optional** and not installed automatically with the extension. Install it only if you use cmux.

See [`skills/cmux-room-reload/SKILL.md`](skills/cmux-room-reload/SKILL.md) for full instructions.

## Current limitations

- File polling instead of a socket daemon.
- Control permission does not persist across full process restarts (new pid = new agent id).
- `reload` and `new_session` via `room_control_agent` are DEPRECATED — pi does not process slash commands from extension-injected messages. Use the optional cmux skill above as a workaround, or ask the user to run `/reload` or `/new` manually.
- No tree/fork orchestration yet.
- No guaranteed exactly-once delivery across process crashes.

These are deliberate MVP trade-offs to keep the extension easy to test and evolve.