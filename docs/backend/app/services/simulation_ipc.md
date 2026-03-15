# `backend/app/services/simulation_ipc.py` — Simulation IPC

## Overview

`simulation_ipc.py` implements the file-based command/response channel between the Flask backend and a running OASIS simulation process. Instead of sockets or message queues, it uses two folders inside a simulation directory:

- `ipc_commands/` for outbound commands from Flask
- `ipc_responses/` for inbound responses written by the simulation process

This keeps cross-process communication simple and easy to debug on a local machine.

---

## Enums

### `CommandType`

| Value | Meaning |
|-------|---------|
| `INTERVIEW` | Ask one agent a question |
| `BATCH_INTERVIEW` | Ask multiple agents in one request |
| `CLOSE_ENV` | Ask the simulation process to shut down cleanly |

### `CommandStatus`

| Value | Meaning |
|-------|---------|
| `PENDING` | Command file written, not yet processed |
| `PROCESSING` | Simulation process picked it up |
| `COMPLETED` | Response generated successfully |
| `FAILED` | Command execution failed |

---

## Data Structures

### `IPCCommand`

Fields:
- `command_id`: UUID string used as both command and response filename
- `command_type`: one of `CommandType`
- `args`: payload for the command
- `timestamp`: ISO creation time

Methods:
- `to_dict()` serializes the command for JSON file writing
- `from_dict()` reconstructs the dataclass from a parsed JSON object

### `IPCResponse`

Fields:
- `command_id`: UUID matching the original command
- `status`: one of `CommandStatus`
- `result`: optional response payload
- `error`: optional error string
- `timestamp`: ISO response time

Methods:
- `to_dict()` serializes the response
- `from_dict()` reconstructs the response object

---

## `SimulationIPCClient`

### `__init__(simulation_dir)`

Creates `ipc_commands/` and `ipc_responses/` inside the simulation directory and stores those paths for later polling.

### `send_command(command_type, args, timeout=60.0, poll_interval=0.5) -> IPCResponse`

Core request/response loop.

Behavior:
1. Generates a UUID command ID.
2. Writes `<command_id>.json` into `ipc_commands/`.
3. Polls `ipc_responses/<command_id>.json` until timeout.
4. Parses the response into `IPCResponse`.
5. Deletes both command and response files after success.
6. Raises `TimeoutError` if no response arrives in time.

This method is the shared primitive used by all higher-level helpers.

### `send_interview(agent_id, prompt, platform=None, timeout=60.0) -> IPCResponse`

Convenience wrapper for a single-agent interview.

Arguments written to the command payload:
- `agent_id`
- `prompt`
- optional `platform`

Uses `CommandType.INTERVIEW`.

### `send_batch_interview(interviews, platform=None, timeout=120.0) -> IPCResponse`

Sends a list of interview payloads in one command using `CommandType.BATCH_INTERVIEW`.

Typical use case: the report agent wants to ask multiple simulation agents related questions without paying the round-trip overhead for each one.

### `send_close_env(timeout=30.0) -> IPCResponse`

Sends a graceful shutdown request with `CommandType.CLOSE_ENV` so the OASIS process can exit cleanly and flush its outputs.

---

## File Protocol

For a simulation directory like `uploads/simulations/sim_xxx/`, the IPC layout is:

```text
ipc_commands/
  <uuid>.json
ipc_responses/
  <uuid>.json
```

The filename is the correlation key. A command is considered complete only when the matching response file appears.

---

## Related Files

- [`services/simulation_runner.py`](simulation_runner.md) uses IPC to interact with a live simulation
- [`services/report_agent.py`](report_agent.md) relies on interview responses indirectly through the tool layer
- [`api/simulation.py`](../api/simulation.md) exposes interview and close-environment endpoints
