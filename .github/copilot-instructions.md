# Copilot instructions for Hyprland Manager

This repository is the Rust workspace for Hyprland Manager, a keyboard-first TUI for managing an Arch Linux + Hyprland desktop. Treat `AGENTS.md` as the canonical project contract and `docs/` as the source of truth for product and architecture details.

## Project shape

- Workspace root: `Cargo.toml`
- Binary crate: `crates/app` (`hyprland-manager`)
- Main TUI crate: `crates/tui`
- Shared types/config: `crates/core`
- Hyprland IPC and event types: `crates/hyprland`
- System and monitoring crates: `crates/system`, `crates/monitoring`, `crates/diagnostics`
- Configuration, backup, and theme flows: `crates/config-manager`, `crates/backup-manager`, `crates/theme-manager`
- AI-related crates: `crates/ai-agent`, `crates/ai-providers`, `crates/ai-tools`, `crates/plugin`
- Storage: `crates/storage`

The app is intentionally modular: the TUI renders and dispatches actions, while the owning crates do the actual OS, config, and IPC work. A simple mental model is `main -> hm_tui::run_with_activity_logs -> run_loop -> ui::draw / handle_key -> AppState`.

## Build, test, and lint

Use these commands from the workspace root (`hyprmanager/`):

```bash
cargo fmt --all -- --check
cargo check --workspace
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace
```

For a single targeted test:

```bash
cargo test -p hm-tui render_dashboard_does_not_panic
```

Run the app locally:

```bash
cargo run -p hyprland-manager
cargo run -p hyprland-manager -- --headless
```

The project’s CI runs the same Rust checks on push and PR; do not treat CI as a replacement for local verification.

## Architecture and behavior to preserve

- The app must remain usable without an AI provider. AI is optional and should degrade gracefully.
- `hm-tui` is UI-focused; it should not directly call OS, systemd, or network layers. Behavior belongs in the owning crate.
- The TUI should stay responsive; long-running work runs on background threads and is polled from the tick loop instead of blocking the event loop.
- Config changes follow a strict order: backup -> validate -> atomic write -> verify/diff. Do not bypass this flow.
- The crate APIs define the central boundaries. Avoid placing direct shell or Hyprland IPC logic in rendering code.
- Plugins are loaded from `~/.local/share/hyprland-manager/plugins/<name>/`, with `plugin.toml` metadata and optional shared libraries. Plugin loading should never abort startup for a failed or missing plugin.

## Project-specific conventions

- Use `hm_core` XDG path helpers for all filesystem paths; do not hardcode `~/.config`, `~/.local/share`, or related paths.
- Keep config file permissions strict: config writes use `0600` and should not loosen that requirement.
- Hyprland JSON is version-variable; preserve `#[serde(default)]` on Hyprland data structs where applicable (for example the window/workspace/monitor types). Missing fields are expected between versions.
- Treat `luac -p` validation as part of the config pipeline. If `luac` is unavailable, fail gracefully and do not pretend the config is valid.
- Never log, print, or commit secrets or provider API keys.
- Destructive actions (restore, delete, theme apply, config write, service restart) must keep their confirmation and backup flow.
- Tests live in `#[cfg(test)]` modules next to the code; cross-crate or smoke-test coverage belongs in `crates/integration-tests/tests`.
- If a behavior change is made, add a focused regression test for it.
- Keep the app functional when Hyprland, systemd, the network, or AI providers are unavailable; degrade with a visible status message, never a panic.

## Useful references

- `AGENTS.md` — canonical development contract and project-specific guardrails
- `CLAUDE.md` — points to `AGENTS.md` and restates the required workflow
- `README.md` — workspace overview and run instructions
- `docs/architecture.md` — crate responsibilities and plugin model
- `docs/development.md` — repo checks and testing structure
- `docs/security.md` and `docs/configuration.md` — safety and config details

## Scope of changes

Keep patches surgical and aligned to the project’s existing boundaries. Favor small, focused fixes over broad refactors. If a change touches config mutation, Hyprland IPC, AI permissions, or destructive flows, verify the surrounding safety behavior explicitly.
