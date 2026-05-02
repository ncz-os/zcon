# zcon — Plan of Record (v0)

**Status:** Alpha / experimental — no code committed yet, this doc is
the design canon to start from.

## Direct lineage

`zcon` is modeled on **Novell NetWare's SYSCON** (specifically the
NetWare 3.x / 4.x version familiar to anyone who admined a small-business
network in the late 80s / early 90s). The reasons:

- **Pane-driven hierarchy.** SYSCON's "Available Topics → sub-list →
  detail form" structure is a clean mental model for fleet admin —
  it scales from a single-device deployment to a multi-device fleet
  without UI rework.
- **Keyboard-first, F-key-anchored.** Every action discoverable from
  the status line. No mouse required (and on edge hardware over SSH,
  often no mouse available).
- **Modal but not slow.** SYSCON's modal forms are surgical — open one,
  do the thing, close. They don't pretend to be live dashboards.
- **Looks intentional.** The Turbo-Vision-style chrome reads as
  "this was designed" not "this was thrown together." Same aesthetic
  zterm v0.3 is targeting.

The Turbo Vision substrate (via `tvision` — the C++ port that's
canonical, or `turbo-vision-4-rust` — the pure-Rust port that zterm
v0.3 is using) gives us:

- TWindow / TDialog / TMenuBar / TStatusLine native primitives
- TInputLine / TButton / TListViewer / TCheckBoxes for forms
- TStreamableClass for save/restore desktop state
- A real desktop with overlapping windows, shadows, accelerator keys

## Top-level pane structure

The main menu mirrors SYSCON's "Available Topics" — hierarchical,
arrow-key driven, F-key shortcuts:

```
┌─ zcon — nclawzero SYSCON ──────────────────────── F1 Help · F10 Menu ─┐
│                                                                       │
│  ┌─ Available Topics ─────────────────────┐                           │
│  │                                        │                           │
│  │  ▶ Agent Information                   │                           │
│  │    Identity & Credentials              │                           │
│  │    Fleet Topology                      │                           │
│  │    Network Configuration               │                           │
│  │    Storage & Filesystems               │                           │
│  │    System Updates                      │                           │
│  │    Audit & Logs                        │                           │
│  │    Supervisor Options                  │                           │
│  │    Quit (F10)                          │                           │
│  │                                        │                           │
│  └────────────────────────────────────────┘                           │
│                                                                       │
└── F1 Help · F2 Save · F3 Insert · F8 Delete · F10 Quit · Esc Cancel ──┘
```

Each top-level pane below describes its sub-list and detail forms.

---

### Pane 1 — Agent Information

Manages the four canonical nclawzero agents: `zeroclaw`, `openclaw`,
`hermes`, `claude-code`. Operations land via the `ncz` CLI under the
hood so all mutations participate in the same install-set.toml +
audit-log discipline.

Sub-list: one row per installed agent, with columns for name, version,
runtime (Podman/Docker), enabled/disabled, last-restart, RSS.

Detail form (Enter on a row):

| Field | Notes |
|---|---|
| Status | running / stopped / failed / not installed |
| Image / Source | OCI ref + digest, or filesystem path |
| Profile | minimal / agent / agent-with-skills (if defined) |
| Variant | per-agent (e.g. zeroclaw v0.7.4) |
| Sandbox | landlock / bwrap / podman / none |
| Quadlet path | `/etc/containers/systemd/<agent>.container` |
| Env file | `/etc/nclawzero/agent-env` (link to "Identity & Credentials" pane) |
| Recent logs | last 50 lines of journalctl for the unit |

Actions (F-keys):
- **F2 Restart** — `systemctl restart <agent>`
- **F3 Install / Reinstall** — calls `ncz agent install`
- **F4 Update** — calls `ncz agent reload` after registry pull
- **F5 Disable** — calls `ncz agent disable`
- **F6 Enable** — calls `ncz agent enable`
- **F7 Logs** — splits to a journalctl-tail viewer pane
- **F8 Uninstall** — calls `ncz agent uninstall <agent>` with dry-run preview
- **F9 Edit env** — opens the agent-env in TInputLine form

---

### Pane 2 — Identity & Credentials

The fleet's secrets-management surface. Operators add/remove API keys,
SSH keys, GPG keys, and the per-agent env-file values without ever
hand-editing a file.

Sub-list (left column): credential category.

| Category | Storage |
|---|---|
| Agent env-file (API keys) | `/etc/nclawzero/agent-env` |
| SSH host keys | `/etc/ssh/ssh_host_*` |
| SSH user authorized_keys | `~/.ssh/authorized_keys` per user |
| GPG keys | `~/.gnupg/` per user |
| TLS / mTLS certs | `/etc/nclawzero/tls/` |
| Cloud provider keys (Together, OpenAI, Gemini, etc.) | linked agent-env entries |

Detail form: editable list of key/value pairs with masked-by-default
view, Reveal toggle, Copy-to-clipboard, Rotate, Audit (when last
rotated, by whom).

Cross-cuts to **Pane 7 — Audit & Logs** so every credential change
emits an audit-log entry.

---

### Pane 3 — Fleet Topology

The "where am I in the fleet" view. zcon talks to the local MNEMOS
client + fleet-auth metadata to show:

- This device's hostname, role, IP, MAC, fleet-internal identity
- List of peer devices (last-seen, role, agents installed)
- MNEMOS sync state (last successful pull/push, drift)
- Cluster-aware actions: "rotate this device's fleet-auth keys",
  "request peer to mirror state from us"

This pane is forward-looking — fleet-auth integration may not exist
yet at v0.1. Initial impl can be read-only (display peers from
`fleet-auth` metadata, no mutations).

---

### Pane 4 — Network Configuration

NetworkManager + nftables wrapper, exposed via SYSCON-style forms.

Sub-list:
- Interfaces (ethernet, wifi, bluetooth, USB tethers)
- Connections (NM-managed)
- Routes (static + default)
- DNS (resolvers, search domains)
- Firewall (nftables rule chains, default policies)
- VPN (WireGuard / Tailscale presence + config)

Detail form per row: editable values + Apply (calls `nmcli` or `nft`
under the hood). Save vs Apply distinction: Save updates config,
Apply also restarts the relevant service.

---

### Pane 5 — Storage & Filesystems

Disk health + capacity + backup config.

Sub-list:
- Block devices (NVMe, USB, SD) with SMART health column
- Mounted filesystems (with usage bar)
- /var/lib/nclawzero — agent state, OCI image cache, ZFS-style snapshot
  history (if ZFS rolls in here later)
- Backup destinations (local snapshot dir, remote rsync target,
  optional cloud sync)
- Swap (if any) — usually none on edge devices

Actions:
- F2 Snapshot now (calls `zfs snapshot` or `restic snapshot`)
- F3 Mount/Unmount
- F4 SMART self-test
- F5 fsck (offline, with reboot warning)

---

### Pane 6 — System Updates

`apt` + `ncz` + agent OCI image versions, all in one place.

Sub-list:
- Debian apt: pending package upgrades count, last update, next scheduled
- ncz CLI: current version vs latest (compares to gitlab API)
- Agent OCI images: per-agent (zeroclaw, openclaw, hermes), current
  digest vs `:latest` digest from registry
- Kernel: running version, available newer version in apt

Actions:
- F2 Update apt index (`apt update`)
- F3 Apply apt upgrades (with dry-run + reboot prompt)
- F4 Update ncz (binary swap)
- F5 Pull latest agent images + restart quadlets
- F6 Kernel install + reboot (with confirmation)

---

### Pane 7 — Audit & Logs

The "what just happened" surface.

Sub-list:
- Recent agent activity (calls, tool invocations, errors) — read from
  agent journals
- Recent system events (auth, kernel oopses, oom-killer)
- Recent zcon mutations — audit log of who did what via this tool
- Recent ncz CLI invocations (separate audit log)
- journalctl tail (live)

Actions:
- F2 Open in pager (less-style scroll)
- F3 Filter by severity / source / time window
- F4 Export to file (for support ticket)

---

### Pane 8 — Supervisor Options

The "are you sure" pane. Locked behind a sudo re-auth even when zcon
itself was launched as root.

Sub-list:
- Change root password
- Change ncz operator password
- Halt / Reboot / Power off
- Factory reset (preserve logs vs full wipe)
- Generate diagnostic bundle (sosreport-style)
- Enable / disable remote support session (opens a scoped tailscale
  tunnel for upstream support)

---

## Implementation phasing

### v0.1 — Read-only scaffold
- Top-level menu working
- Panes 1, 5, 7 in read-only mode (can browse agents, storage, logs)
- All other panes: stub with "Not implemented in v0.1" placeholder
- Saves desktop state via TStreamableClass on quit
- Smoke test: `zcon` launches, navigates, quits cleanly on Pi 4 + cixmini

### v0.2 — Agent mutations
- Pane 1 (Agent Information) full read/write — calls `ncz` under the hood
- Pane 2 (Identity & Credentials) read/write for agent-env-file
- Pane 8 (Supervisor) basic actions (reboot, halt, password change)
- Audit-log integration: every mutation in v0.2 writes to `/var/log/zcon-audit.log`

### v0.3 — Network + storage mutations
- Pane 4 (Network) full read/write via NetworkManager D-Bus + nft
- Pane 5 (Storage) full read/write incl. snapshots, fsck

### v0.4 — Fleet awareness
- Pane 3 (Fleet Topology) read-only peer list
- Cross-fleet rotation actions (push key changes to peers)
- MNEMOS bidirectional sync from this pane

### v0.5+ — Polish, theming, accessibility
- Light/dark theme toggle (matching zterm v0.3)
- Multi-language layout
- Screen-reader compatibility hooks (aria-label-equivalent for tvision)

## Dependencies on sister projects

- **zterm v0.3** — uses the same Turbo Vision substrate, but a different
  application. Don't share runtime; do share the build-time tvision
  configuration so chrome looks identical (consistent F-key palette,
  status-line idiom, color theme).
- **ncz-tools** — zcon's Pane 1 mutations call `ncz agent ...` subcommands
  via subprocess. This means ncz-tools must remain a CLI-with-stable-flags
  contract; zcon depends on that surface.
- **MNEMOS** — Pane 3 reads MNEMOS sync state via the local MCP socket.
  Read-only at v0.4; mutating Pane 3 actions arrive in v0.5+.
- **fleet-auth** — Pane 3 reads peer list from the fleet-auth registry.
  Same deferred-write story as MNEMOS.

## Non-goals

- **No mouse-required interactions.** Mouse may pan the desktop but
  every action must be reachable via keyboard.
- **No web UI.** Per the [zterm doctrine](https://gitlab.com/perlowja/zterm)
  ("Back to the 1970s Rust: light, thin, fast, memory-safe. API is
  authoritative. Local or network. No web junk.") — zcon inherits the
  same posture.
- **No platform abstraction.** This is a Linux-only tool; the system
  primitives (NetworkManager, nftables, systemd, journalctl, apt,
  podman) are not pluggable.
- **No graphical session.** Runs in any tty including SSH; no X11 or
  Wayland dependency.

## Open design questions

- **Rust vs C++?** zterm v0.3 is exploring `turbo-vision-4-rust` (pure
  Rust port). zcon could:
  - (a) Use the same Rust port — matches zterm's substrate, simpler
    cross-project tooling, but the Rust port may lag the C++ original.
  - (b) Use C++ `tvision` directly — production-tested, full feature
    set, but adds a C++ build to a Rust-leaning fleet.
  - **Tentative direction**: track zterm v0.3's decision. If
    turbo-vision-4-rust ships the panes we need, use it.
- **Single binary or modular?** A single static binary is simplest to
  install (one `apt install zcon`); modular plugins per-pane is more
  extensible. v0.1 lands single-binary; revisit if/when third-party
  pane authors emerge.
- **Audit log format?** Plain text vs JSON-lines. JSON-lines is easier
  to query later; plain text is human-readable. Probably JSON-lines
  with a "tail-friendly" pretty-printer in Pane 7.

## References

- Novell NetWare SYSCON docs (NetWare 3.12 / 4.11): preserved at
  https://archive.org/details/netware-3-12-utilities-reference
- Turbo Vision (Borland 1991+) original architecture and the modern
  ports:
  - https://github.com/magiblot/tvision (canonical C++ port, MIT)
  - https://github.com/SerenityOS/serenity (uses Turbo Vision-style chrome
    in TextEditor and Spreadsheet)
  - turbo-vision-4-rust (the pure-Rust port zterm v0.3 is using)
- nclawzero internal: `~/.claude/rules/handoff-2026-04-30-convergence-wrapper-ncz.md`
  for ncz-tools install-set.toml architecture (zcon Pane 1 calls into this)
