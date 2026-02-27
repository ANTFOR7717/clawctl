# Clawctl

Clawctl is a ZeroClaw agent orchestration tool that manages multiple ZeroClaw agent instances as service units. It is built on top of [rustysd](https://github.com/KillingSpark/rustysd), a systemd-compatible service manager written in Rust by Moritz Borcherding.

Each ZeroClaw agent runs as a clawctl service unit with an isolated workspace and configuration. Clawctl handles agent lifecycle, dependency ordering, socket activation, and runtime control.

## Architecture

See [doc/Architecture.md](doc/Architecture.md) for a full description of the inherited rustysd architecture and planned ZeroClaw extension points.

## Quick Start

```bash
# Build
cargo build --release

# Run with default config (uses config/clawctl_config.toml and test_units/)
./target/release/clawctl

# Control interface
./target/release/clawctl-ctl /path/to/notifications/control.socket list-units
./target/release/clawctl-ctl /path/to/notifications/control.socket status agent-01.service
./target/release/clawctl-ctl /path/to/notifications/control.socket restart agent-01.service
```

## Build Options

```bash
# Default build
cargo build --release

# With Linux cgroup support
cargo build --release --features cgroups

# With DBus service type support
cargo build --release --features dbus_support

# With eventfd (Linux only, more efficient than pipes)
cargo build --release --features linux_eventfd
```

## Configuration

Config is loaded from `config/clawctl_config.toml` by default, or from a directory specified with `--conf`:

```toml
logging_dir = "./logs"
log_to_stdout = true
log_to_disk = false
notifications_dir = "./notifications"
unit_dirs = [ "./test_units" ]
target_unit = "default.target"
```

Environment variables prefixed with `CLAWCTL_` override config file values.

## Unit Files

Unit files follow the systemd INI format and live in the `unit_dirs` directories. See `test_units/` for examples.

```ini
[Unit]
Description=Example ZeroClaw Agent
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/zeroclaw --config /etc/zeroclaw/agent-01.toml
Restart=on-failure

[Install]
WantedBy=default.target
```

## Upstream: Rustysd

This project forks [KillingSpark/rustysd](https://github.com/KillingSpark/rustysd) (MIT License, Copyright 2019 Moritz Borcherding). All original rustysd functionality is preserved intact. See [doc/Architecture.md](doc/Architecture.md) for full documentation of inherited capabilities.

Future milestones (MAR-30, MAR-31) will extend this base to add the `zeroclaw-agent` unit type and full agent lifecycle management.

## License

MIT — see [LICENSE](LICENSE).
