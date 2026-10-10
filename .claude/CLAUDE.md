# CLAUDE.md

Guidance for Claude (and anyone else) working in this repo. This is a local-only
Flutter dashboard for a legacy UniFi network, talking to the devices directly
over SSH — no cloud, no controller at runtime.

## Project Overview

- **Hardware**: UniFi Security Gateway (USG), an 8-port UniFi PoE switch, and a
  UniFi AP, all the same generation. The USG is EOL (Nov 2024); firmware is the
  4.x EdgeOS line.
- **Network**: LAN is `192.168.0.0/24`, gateway `192.168.0.1`.
- **Controller**: A UniFi Network Application stack (Docker Compose,
  `lscr.io/linuxserver/unifi-network-application` pinned to **8.6.9** — the last
  version that can push config to a USG — plus `mongo:7.0`) runs **on demand
  from a laptop** for initial adoption and occasional reconfiguration only. It
  is never a runtime dependency of the app. Do not design any app feature that
  assumes the controller is reachable.
- **No cloud, no remote access.** Tailscale, WireGuard, and L2TP were all
  considered and explicitly rejected in favor of local-only. Don't reintroduce
  them without a new decision to do so.

## What the App Does (v1 scope)

Read-only dashboard, iOS + Android. First screen is WAN + system summary; next
are client list and throughput. **No config writes or saves in v1** — this was
a deliberate scope cut to avoid EdgeOS's commit-lock and controller-overwrite
problems. Don't add config-writing code paths without discussing scope first.

## How the App Talks to the Network

The USG has no local HTTP API. Everything goes over SSH using EdgeOS's own
command wrappers:

- `vyatta-op-cmd-wrapper` — operational (`show ...`) commands.
- `vyatta-cfg-cmd-wrapper` — config commands. **Unused in v1.**
- `mca-dump` — the preferred data source. Returns the same JSON payload the
  device reports to the controller (system health, WAN status, interface
  counters). Its exact keys vary by firmware build — treat any field name as a
  guess until it's been confirmed against **this device's** captured output,
  not general UniFi documentation.
- `show dhcp leases` + `show arp` — client discovery fallback/supplement,
  merged into one "devices on the network" view (leases alone aren't honest
  about who's actually present).

## Repo Layout

Dart pub workspace (Dart 3.6+), one lockfile, two packages:

```ASCII
usg-dashboard/
├─ pubspec.yaml          # workspace root
├─ docs/adr/             # architecture decision records
├─ packages/
│  └─ usg_client/        # pure Dart — NO Flutter dependency
│     ├─ lib/
│     │  ├─ usg_client.dart   # public API
│     │  ├─ testing.dart      # fake CommandRunner, shared with app tests
│     │  └─ src/
│     │     ├─ transport/     # CommandRunner interface + dartssh2 impl
│     │     ├─ commands/      # builds wrapped EdgeOS commands
│     │     ├─ parsers/       # DTOs + parsers, tested against fixtures
│     │     ├─ models/        # GatewaySnapshot, WanStatus, etc.
│     │     ├─ rates/         # RateTracker
│     │     └─ errors.dart    # sealed error hierarchy
│     ├─ test/fixtures/       # real captured device output
│     └─ bin/bench.dart       # CLI benchmark against the real USG
└─ apps/dashboard/            # Flutter
   └─ lib/
      ├─ app/                 # theme, routing
      ├─ core/                # secure storage, lifecycle, formatters
      └─ features/
         ├─ dashboard/{state,widgets}
         └─ settings/
```

## Hard Rules

These are decisions already made, not defaults to reconsider mid-task:

- **Dependency direction: `app → package`, never the reverse.** Nothing in
  `usg_client` may import anything Flutter.
- **`dartssh2` is only imported inside `usg_client`.** Every SSH call goes
  through the `CommandRunner` interface — no widget or app-layer code opens a
  socket directly.
- **Minimal dependencies is a security requirement, not a style preference.**
  Approved runtime deps today: `dartssh2`, `flutter_secure_storage`. Adding
  anything else needs an ADR in `docs/adr/` justifying it — don't reach for a
  package as a shortcut.
- **No Riverpod, no Bloc.** State management is plain Flutter:
  `ChangeNotifier`, `ValueNotifier`, `AppLifecycleListener`. This was a
  deliberate rejection, not an oversight.
- **Never hand-roll SSH or crypto.** `dartssh2` stays the transport as-is;
  don't "simplify" by reimplementing protocol pieces.
- **One short-lived SSH connection per refresh cycle**: connect, run the
  batched commands, disconnect. No persistent/pooled connections — iOS kills
  backgrounded sockets, so persistence buys nothing and adds failure modes.
  Poll every 5–10s while foregrounded; pause via lifecycle hooks when
  backgrounded.
- **Models are tolerant by design**: nullable fields, unknown keys ignored,
  never fatal on an unexpected shape. `mca-dump`'s shape changes across
  firmware builds.
- **Rate math**: USG counters are cumulative byte counts, not throughput.
  `RateTracker` keeps the previous sample + timestamp per counter and computes
  delta/sec; a counter that decreases (reboot or wrap) is a reset, never a
  negative rate.
- **Errors are a sealed hierarchy** so UI code can `switch` over them
  exhaustively: unreachable, auth failed, host key mismatch, command failed,
  commit failed (not reachable in v1), parse failure.
- **Formatting lives in the app, not the package.** `usg_client` returns typed
  values (`Duration`, byte counts); turning those into "12d 4h" or "42.3 Mbps"
  is a `core/` concern in `apps/dashboard`.

## Testing

- Unit tests (both package and app) use the fake `CommandRunner` exported from
  `usg_client/testing.dart` — never real SSH in the automated suite.
- Parsers are tested against real captured fixtures in `test/fixtures/`
  (`mca-dump`, `show dhcp leases`, `show arp`). Capture actual output from this
  device before trusting a parser against it.
- `bin/bench.dart` is a standalone CLI (pure Dart, reuses `usg_client`) for
  timing handshake vs. command cost against the real USG. It's a manual tool,
  not part of `dart test`.

## Security Notes

- Credentials go through `flutter_secure_storage` only.
- SSH login for the phone should use a dedicated EdgeOS user at
  `level operator` (read-only) rather than the admin account, so a lost or
  stolen phone can't reconfigure the network.
- Host key mismatches are a distinct, explicit error case — never
  accept-and-continue silently.

## Explicitly Out of Scope Right Now

- Any config writes/commits to the USG.
- Cloud hosting or remote access of any kind.
- Anything that assumes the controller is running at app runtime.

## In Progress / Not Yet Started

- Capturing real `mca-dump`, `show dhcp leases`, and `show arp` fixtures from
  the actual USG.
- Drafting the minimal-dependencies ADR.
- Designing the `CommandRunner` interface.
- Switch and AP clients haven't been started — the USG comes first; treat the
  three-device merge layer as a later milestone, not a v1 requirement.
