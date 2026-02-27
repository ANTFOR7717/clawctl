# Clawctl Architecture

Clawctl is a ZeroClaw agent orchestration tool that manages multiple ZeroClaw agent instances as service units. It is built on top of [rustysd](https://github.com/KillingSpark/rustysd), a systemd-compatible service manager written in Rust.

## Upstream: What Rustysd Provides

### Unit Type System

Rustysd provides three unit types, each parsed from INI-format files:

| Type | File Extension | Purpose |
|------|----------------|---------|
| Service | `.service` | A managed process with lifecycle hooks |
| Socket | `.socket` | Socket activation config (binds to a service by name) |
| Target | `.target` | Synchronization point / milestone (like a dependency group) |

Each unit file has standard sections:
- `[Unit]` — metadata, `Description=`, `After=`, `Before=`, `Wants=`, `Requires=`
- `[Service]` — process config, `ExecStartPre=`, `ExecStart=`, `ExecStartPost=`, `ExecStop=`, `ExecStopPost=`, `Type=`, `User=`, `Group=`, `Environment=`
- `[Install]` — `WantedBy=`, `RequiredBy=`

### Process Lifecycle Management

Service types supported: `simple` (default), `notify` (waits for `READY=1` sd_notify), `dbus` (optional, waits for DBus name), `oneshot`.

Lifecycle hooks in execution order:
1. `ExecStartPre` — run before starting
2. `ExecStart` — the main process
3. `ExecStartPost` — run after start
4. `ExecStop` — run to stop the service
5. `ExecStopPost` — run after stop

### JSON-RPC 2.0 Control Interface

Clawctl exposes a JSON-RPC 2.0 control socket at `<notifications_dir>/control.socket` (Unix domain socket).

The `clawctl-ctl` binary is a thin CLI client that serializes commands and pretty-prints responses.

**Available Commands:**

| Method | Params | Description |
|--------|--------|-------------|
| `list-units` | optional kind: `"service"`, `"socket"`, `"target"` | List all loaded units |
| `status` | optional unit name | Show unit status |
| `start` | unit name | Start a unit |
| `start-all` | unit name | Start unit and all dependencies |
| `stop` | unit name | Stop a unit |
| `stop-all` | unit name | Stop unit and all dependents recursively |
| `restart` | unit name | Restart a unit |
| `enable` | unit name(s) | Load new unit file(s) at runtime |
| `reload` | none | Reload all new units from unit dirs |
| `reload-dry` | none | Dry-run reload (parse only) |
| `remove` | unit name | Remove a unit |
| `shutdown` | none | Gracefully shut down clawctl |

**Example usage:**
```bash
clawctl-ctl /run/clawctl/control.socket list-units service
clawctl-ctl /run/clawctl/control.socket status agent-01.service
clawctl-ctl /run/clawctl/control.socket restart agent-01.service
```

### Signal Handling

Clawctl handles OS signals:
- `SIGCHLD` — reap exited child processes, trigger restart/dependency logic
- `SIGTERM` / `SIGINT` / `SIGQUIT` — initiate graceful shutdown

### Socket Activation

Services can be started lazily when their socket receives a connection. Socket file descriptors are passed to the service process at index 3, 4, 5, ... (sd_notify convention).

### Cgroup Support (Optional)

When built with `--features cgroups`, clawctl uses Linux cgroups to:
- Track all processes belonging to a service (even forked children that change process group)
- Reliably kill the entire process tree on stop

### Dependency Graph

Units declare ordering via `After=` / `Before=` and membership via `Wants=` / `Requires=`. At startup, clawctl:
1. Loads and parses all unit files from configured `unit_dirs`
2. Resolves dependencies and prunes to only what is needed for `target_unit`
3. Performs a cycle-detection sanity check
4. Activates units in parallel where dependencies allow

### Runtime Architecture

Central shared state: `Arc<RwLock<RuntimeInfo>>` containing:
- `unit_table: HashMap<UnitId, Unit>` — all loaded units and their state
- `pid_table: Mutex<HashMap<Pid, PidEntry>>` — running processes
- `fd_store: RwLock<FDStore>` — open socket file descriptors
- `config: Config` — runtime config

Background threads:
1. **Signal handler** — SIGCHLD/SIGTERM/SIGINT/SIGQUIT
2. **Control socket acceptor** — JSON-RPC 2.0 listener (one thread per connection)
3. **Notification handler** — reads sd_notify messages from services
4. **Stdout handler** — reads/buffers service stdout
5. **Stderr handler** — reads/buffers service stderr
6. **Socket activation** — `select()` loop on socket FDs, activates services on connection

### Binary Architecture

Two binaries (plus test utilities):

| Binary | Source | Purpose |
|--------|--------|---------|
| `clawctl` | `src/bin/clawctl.rs` | Main service manager daemon (also serves as `exec_helper` when invoked by that name) |
| `clawctl-ctl` | `src/bin/clawctl_ctl.rs` | Control client — sends JSON-RPC 2.0 commands to the daemon |

The `clawctl` binary is dual-purpose: it inspects `argv[0]` — if invoked as `exec_helper`, it reads an `ExecHelperConfig` from stdin and `execv()`s into the service binary (used for privilege dropping); if invoked as `clawctl` it runs the service manager.

### Configuration

Config is loaded from (in priority order):
1. `CLAWCTL_*` environment variables (e.g. `CLAWCTL_UNIT_DIRS`, `CLAWCTL_TARGET_UNIT`)
2. `<config_dir>/clawctl_config.toml`
3. `<config_dir>/clawctl_config.json`
4. Built-in defaults

**Default config (`config/clawctl_config.toml`):**
```toml
logging_dir = "./logs"
log_to_stdout = true
log_to_disk = false
notifications_dir = "./notifications"
unit_dirs = [ "./test_units" ]
target_unit = "default.target"
```

**CLI flags for `clawctl`:**
```
-c, --conf <PATH>   Path to config directory
-d, --dry-run       Parse and validate units, then exit
```

---

## Clawctl Extension Points

The following areas are designed for extension in future milestones:

### ZeroClaw Agent Unit Type (MAR-30)
A new unit kind `agent` (`.agent` files) that wraps a ZeroClaw instance with:
- Isolated workspace directory per agent
- Per-agent config directory
- Agent-specific environment variables (`ZEROCLAW_AGENT_ID`, `ZEROCLAW_WORKSPACE`, etc.)

### Agent Lifecycle Management (MAR-31)
- Agent provisioning: workspace creation, config injection
- Agent status reporting: maps ZeroClaw internal state to unit status
- Multi-agent coordination: dependency ordering between agents

---

## Source Layout

```
src/
├── lib.rs                     # Crate root
├── config.rs                  # Config loading (TOML/JSON/env)
├── runtime_info.rs            # Central shared state (RuntimeInfo)
├── signal_handler.rs          # OS signal handling
├── socket_activation.rs       # Socket activation loop
├── notification_handler.rs    # sd_notify + stdout/stderr handling
├── shutdown.rs                # Graceful shutdown
├── fd_store.rs                # File descriptor store
├── logging.rs                 # Logging setup (fern)
├── dbus_wait.rs               # DBus service readiness (optional)
│
├── bin/
│   ├── clawctl.rs             # Main daemon entry point
│   ├── clawctl_ctl.rs         # Control client
│   ├── testservice.rs         # Test service for development
│   └── testserviceclient.rs   # Test client for development
│
├── control/
│   ├── control.rs             # Command enum, execute_command(), socket acceptors
│   └── jsonrpc2.rs            # JSON-RPC 2.0 types
│
├── entrypoints/
│   ├── service_manager.rs     # run_service_manager(): startup pipeline
│   └── exec_helper.rs         # run_exec_helper(): post-fork exec with privilege drop
│
├── platform/
│   ├── drop_privileges.rs     # setuid/setgid
│   ├── eventfd.rs             # EventFd abstraction
│   ├── subreaper.rs           # prctl(PR_SET_CHILD_SUBREAPER)
│   ├── unix_common.rs         # Unix helpers
│   ├── grnam.rs               # getgrnam_r wrapper
│   ├── pwnam.rs               # getpwnam_r wrapper
│   └── cgroups/               # Linux cgroup v1/v2 support (feature-gated)
│
├── services/
│   ├── services.rs            # Service struct, start/kill
│   ├── start_service.rs       # Service start logic
│   ├── prepare_service.rs     # Pre-start preparation
│   ├── fork_child.rs          # Child-side post-fork setup
│   ├── fork_parent.rs         # Parent-side post-fork tracking
│   ├── fork_os_specific.rs    # OS-specific post-fork
│   ├── kill_os_specific.rs    # OS-specific kill
│   └── service_exit_handler.rs# Exit handling: restart/notify dependents
│
├── sockets/
│   ├── mod.rs                 # Socket struct, open/close, SocketKind
│   ├── unix_sockets.rs        # Unix domain socket configs
│   ├── network_sockets.rs     # TCP/UDP socket configs
│   └── fifo.rs                # Named pipe configs
│
├── units/
│   ├── id.rs                  # UnitId, UnitIdKind
│   ├── status.rs              # UnitStatus enum
│   ├── unit.rs                # Unit, Common, Specific, dependencies
│   ├── from_parsed_config.rs  # Parsed config → Unit structs
│   ├── loading/               # load_all_units(), dependency resolving
│   ├── unit_parsing/          # INI parser for .service/.socket/.target
│   └── unitset_manipulation/  # activate, deactivate, insert, remove, locking
│
└── tests/
    ├── ordering.rs            # Dependency ordering tests
    ├── parsing.rs             # Unit file parsing tests
    └── state_transition.rs    # Unit state machine tests
```
