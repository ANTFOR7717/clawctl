# Rustysd Architecture — Clawctl Foundation

This document describes the Rustysd components that form the foundation of Clawctl,
a ZeroClaw agent orchestration tool. Clawctl manages multiple ZeroClaw agent instances
as service units using Rustysd's unit lifecycle engine.

## What Rustysd Provides

### Unit Type System

Rustysd operates on "units" — discrete managed entities of three types:

- **Service** (`.service`) — A managed process with lifecycle (start/stop/restart)
- **Socket** (`.socket`) — A file descriptor source for socket activation
- **Target** (`.target`) — A synchronization point / dependency group

Unit files use INI format with sections:
- `[Unit]` — Description, dependencies (`Requires=`, `After=`, `Before=`, `Wants=`)
- `[Service]` — Execution config (`ExecStart=`, `ExecStop=`, `Type=`, `User=`, `Environment=`)
- `[Install]` — Enablement (`WantedBy=`, `RequiredBy=`)

### JSON-RPC 2.0 Control Interface

Rustysd exposes a control socket (TCP or Unix domain) accepting JSON-RPC 2.0 commands:

| Method | Description |
|--------|-------------|
| `list-units` | List all loaded units and their status |
| `status` | Get status of a specific unit |
| `restart` | Restart a unit |
| `stop` | Stop a unit |
| `enable` | Enable (add) a new unit at runtime |
| `shutdown` | Shut down rustysd and all managed services |
| `reload` | Reload unit configuration |

The `rsdctl` binary is a thin CLI wrapper that serializes commands to JSON-RPC 2.0
and sends them to the control socket.

### Process Lifecycle Management

Service lifecycle hooks (executed in order):

1. `ExecStartPre` — Pre-start commands (run as shell commands)
2. `ExecStart` — Main service command (the managed process)
3. `ExecStartPost` — Post-start commands
4. `ExecStop` — Stop commands
5. `ExecStopPost` — Post-stop cleanup

Service types:
- `simple` — Process started directly; considered ready immediately
- `notify` — Process sends `READY=1` via sd_notify socket when ready
- `oneshot` — Single-run command; considered done when process exits

### Signal Handling

Rustysd handles Unix signals:
- `SIGCHLD` — Child process exit; triggers service exit handling
- `SIGTERM` / `SIGINT` / `SIGQUIT` — Graceful shutdown of all managed units

### Socket Activation

Services with associated `.socket` units are started lazily — the socket is opened
immediately but the service only starts when a connection arrives. This enables fast
startup by deferring service initialization until needed.

### Dependency Resolution

Unit startup is ordered by `After=`/`Before=` directives. Unrelated units start in
parallel. Circular dependencies are detected and reported at startup.

### Optional Platform Features

| Feature Flag | Description |
|--------------|-------------|
| `linux_eventfd` | Use Linux eventfds instead of pipes for select() interruption |
| `cgroups` | Use cgroups v1/v2 to reliably track and kill all processes of a service |
| `dbus_support` | Support for `Type=dbus` services waiting for D-Bus name registration |

## Clawctl Extension Points

Future issues (MAR-30, MAR-31) will extend this base to add:

1. **`zeroclaw-agent` unit type** — A specialized service unit type that:
   - Provisions an isolated workspace directory per agent
   - Manages per-agent configuration files
   - Handles agent-specific environment variables
   - Supports agent lifecycle (spawn, health-check, drain, terminate)

2. **Agent orchestration commands** — Extensions to the JSON-RPC control interface:
   - `spawn-agent` — Create and start a new ZeroClaw agent instance
   - `list-agents` — List all running agent instances with status
   - `drain-agent` — Gracefully drain an agent before stopping
   - `assign-task` — Route a task to an available agent

## Binary Overview

| Binary | Purpose |
|--------|---------|
| `clawctl` | Main service manager daemon (renamed from `rustysd`) |
| `rsdctl` | Control client — sends JSON-RPC 2.0 commands to clawctl |
| `testservice` | Test service for socket activation validation |
| `testserviceclient` | Test client for socket activation validation |

## Source Structure

```
src/
├── lib.rs                    — Crate root
├── config.rs                 — Configuration loading (clawctl_config.toml)
├── control/                  — JSON-RPC 2.0 control interface
├── entrypoints/              — Binary entrypoints (service_manager, exec_helper)
├── platform/                 — OS-specific: cgroups, eventfd, subreaper, privileges
├── services/                 — Service start/stop/fork logic
├── sockets/                  — Socket type implementations (TCP, Unix, FIFO)
├── units/                    — Unit type system, parsing, dependency resolution
└── tests/                    — Integration tests: ordering, parsing, state transitions
```
