# zcon — nclawzero SYSCON

**Multi-pane administrative console for the nclawzero distro.**

Inspired by [Novell NetWare's SYSCON](https://en.wikipedia.org/wiki/SYSCON_\(NetWare\)) —
the IBM-character-mode TUI that managed every aspect of a NetWare server
(users, groups, file system rights, server config, accounting) through
hierarchical menus and modal forms.

`zcon` is the equivalent for nclawzero: **one TUI to administer every
aspect of multi-agent fleet configuration** on a single device — agents,
identities, credentials, network, storage, fleet topology, updates, logs,
supervisor actions.

Built on the same [Turbo Vision](https://github.com/magiblot/tvision)
substrate as [zterm v0.3](https://gitlab.com/perlowja/zterm).

---

## ⚠️ Status: Alpha / experimental

> **This project has not yet had significant code committed.** It exists
> here as a positioned plan-of-record alongside its sibling projects
> [zterm](https://gitlab.com/perlowja/zterm) and
> [ncz-tools](https://gitlab.com/nclawzero/ncz-tools), which are also
> alpha and unfinished.
>
> Interfaces, file formats, screen layouts, and even the project name
> may change without notice. **Do not depend on this for anything
> beyond exploration.**

---

## What it covers

`zcon` is invoked as `sudo zcon` on a running nclawzero device and
provides a single TUI entry point for everything an operator might
otherwise need to do via separate tools (`ncz`, `nmcli`, `systemctl`,
`journalctl`, `apt`, hand-edited config files).

See [`docs/PLAN.md`](docs/PLAN.md) for the full pane-by-pane design.

## Design principles

1. **One screen, one job** — each pane is a complete admin task surface.
   No cross-pane state, no clicks lost in tab-driven dialogs.
2. **Keyboard-first** — every action is reachable from arrow keys + enter
   + F-keys. Mouse support is incidental, never required.
3. **Dry-run before destructive** — every write operation shows what it
   will change before applying. Modeled after `kubectl --dry-run`.
4. **Operator's mental model is fleet-aware** — actions taken on one
   device that should propagate (e.g., key rotation, MNEMOS sync) are
   surfaced as such, not hidden behind opaque commands.
5. **Reads are free, writes are gated** — anyone with shell access can
   inspect; mutations require confirmation + audit-log entry.
6. **Composes with `ncz`, not replaces it** — zcon is a TUI front-end;
   the underlying mutations call `ncz` subcommands so scripted operators
   and zcon-driven operators converge on the same audit trail.

## Sister projects

- [`gitlab.com/perlowja/zterm`](https://gitlab.com/perlowja/zterm)
  — Rust TUI for chatting with the agents (different surface: zterm
  is for *talking to* the agents at runtime; zcon is for *configuring*
  the agents and the system around them).
- [`gitlab.com/nclawzero/ncz-tools`](https://gitlab.com/nclawzero/ncz-tools)
  — `ncz` CLI for installing/managing agents on Pi/cixmini fleet
  devices. zcon's mutation paths call `ncz` under the hood for
  agent-lifecycle operations.
- [`gitlab.com/nclawzero/cix-installer`](https://gitlab.com/nclawzero/cix-installer)
  — Debian-installer ISO that produces an nclawzero distro on Cix Sky1
  hardware. Once installed, `sudo zcon` is the canonical admin path.

## License

Apache-2.0
