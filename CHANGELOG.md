# Changelog

All notable changes to lambe will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Docs: a generated configuration reference (`docs/reference/config.md`, checked by a Go test so it cannot drift from the code), a real contributor `docs/ARCHITECTURE.md` (the planning-era document moved to `docs/research/`), `docs/TROUBLESHOOTING.md`, `docs/GLOSSARY.md`, `docs/UPGRADING.md`, and a documentation index in the README.
- `lambe completion bash|zsh|fish` prints shell completions generated from the help text. The Homebrew formula installs them, the deb/rpm/apk packages ship them under the standard vendor completion directories, and the release archives include a `completions/` directory.

### Changed

- Releases now sign every artifact (archives, deb/rpm/apk packages, SBOMs and `checksums.txt`) with cosign, one `<file>.sigstore.json` bundle each, instead of only `checksums.txt`; verification steps are in the README and docs/RELEASING.md. Release binaries and packages are also reproducible from the commit.

- Rex palette actions now raise failures with the reason (lambe not found, no runs yet, a picker that could not show) instead of only logging them, and say once that the pickers need Rex preview.21 (API revision 17) instead of silently opening the dashboard on older servers. `lambe setup rex` now installs a small managed `lambe.lua` shim plus a `lambe/` module directory, and creates `lambe.config.lua` (with the absolute `lambe` path) for your settings so editing them no longer makes updates refuse. Re-run `lambe setup rex --apply` to migrate an older single-file install (an unedited one updates cleanly; if you edited its `override` variables, move them into `lambe.config.lua` and pass `--force`).
- CLI help and errors: `lambe --help`, `-h`, `help [COMMAND]` and `--version` print to stdout and exit 0; every command has `-h` usage with examples; usage errors exit 2 with a one-line reason and a `run 'lambe help CMD'` hint (unknown commands say so); one-shot commands print `lambe: message` instead of JSON log lines (`run` and `mcp` keep structured logs). Exit codes are documented in docs/REFERENCE.md.
- `lambe doctor --label NAME` replaces `-service-label` (still accepted, prints a deprecation note). `LAMBE_CONFIG` sets the default `--config`.

### Fixed

- Docs: removed the stale duplicated "Reference" half of `docs/OPERATIONS.md`, the configuration tables that named keys that do not exist (`tick`, `sync_interval` / `LAMBE_SYNC_INTERVAL`), and the broken `#os-level-restart-production` anchor. The README mascot image is 124 KB instead of 795 KB.

- **`lambe init` no longer writes outside the directory you ran it in.** It used to adopt any `.beads` found in a parent directory, including a global `~/.beads`, and write `WORKFLOW.md` and a starter formula there. It now ignores stub `.beads` directories, uses a real ancestor tracker only with `--tracker PATH` (or an interactive confirmation that names the path; never for `$HOME`), and `--non-interactive` refuses with a clear message.
- Unknown keys in the config file are now an error with a "did you mean" suggestion, and a value of the wrong type names the bad input (`supervisor.attempt_cap: expected an integer, got "banana"`) instead of silently reading 0.
- `lambe status -json` (and the state API) emit `[]`, never `null`, for empty `runs`, `needs_human` and `gate_rejections`.

### Added

- Rex: run output can open as a split beside the pane you ran the action from (`placement=split` or `placement = "split"` in `lambe.config.lua`), an already-open run tab is found in any session on the server instead of being duplicated, and run tabs stay open after the run finishes so its output stays readable.
- Rex: **Lambe: Needs You…** lists only the runs waiting on a human. **Open Run…** now sorts needs-you runs first (`▲`), shows plain-language phases, hides `attempt 0 · turn 0`, counts workflow runs per distinct issue, and explains empty lists (stopped daemon, no runs, unknown workflow).
- A luajit test suite for the Rex integration (`luajit contrib/rex/test/run.lua`, CI job `rex-lua`) with a strict `rex` stub, version profiles for Rex preview.16, .19, and .21, and golden picker rows. `lambe_check` now returns a machine-readable report (plugin version, Rex API revision, picker availability, binary, doctor output) and fails when `lambe doctor` fails.

### Removed

- `contrib/rex/check.lua`, replaced by the hermetic suite and the live `lambe_check` action.

## [0.11.1] - 2026-11-22

### Fixed

- `lambe doctor` no longer reports a finished run (phase `stopped: ...`, which the daemon keeps in its snapshot) as silent or unfinished, so it no longer fails with "the watchdog should have reclaimed it" long after a run completed.

## [0.11.0] - 2026-11-22

### Added

- `lambe setup rex [--apply] [--uninstall] [--force] [--dir DIR]` installs the Rex palette actions (`contrib/rex/lambe.lua`). It is a dry run by default, writes only a header-marked `lambe.lua`, never edits your `init.lua` (it prints the snippet and a suggested key binding instead), and refuses to overwrite a file you modified without `--force`.
- Rex: run tabs get a pane title `ISSUE · Workflow · state` (refreshed when you reopen the run, never over a title you changed), and the Open Run picker marks each row `●` live, `◌` stopped daemon, or `○` archived, live runs first.
- `lambe status -json` prints the morning report as a state API snapshot (`GET /api/v1/state` shape) read from the tracker, with no daemon needed; `lambe state-dir [-json]` prints the resolved state directory and the live state API descriptors (never the token).
- `lambe service install|status|uninstall -label NAME` and `lambe doctor -service-label NAME` run a second, isolated launchd/systemd service; `service install` bakes in `LAMBE_STATE_DIR` when set.

- `supervisor.orphan_claim_ttl` (default 1h; 0 keeps the old behavior) stops heartbeating a recovered claim that has no continuation state once its recorded agent is provably gone, so the lease expires on its own. `lambe claims list` / `lambe claims release [--force] ISSUE` and an "orphaned claims" section in `lambe status` surface them, `lambe doctor` warns about them, and release only ever happens on explicit operator action.

- `supervisor.bd_timeout`, `bd_write_timeout`, `bd_max_concurrency`, and `bd_queue_wait` tune the `bd` subprocess limits that were a fixed 60s per call with no cap. The per-call timeout now counts only the process run, not the wait for a slot. `lambe doctor` times a trivial `bd list` and a metadata query on an embedded hub and warns when either is slow, recommending Dolt server mode. See `docs/solutions/slow-embedded-dolt.md`.

### Changed

- `bd` subprocesses are capped (default 3) behind a priority queue: writes (claim, attempt record, close) are served before reads and polls, which are served before heartbeats, so a backlog of heartbeats can no longer delay a claim's attempt record and keep the agent from being prompted.
- Recording an attempt costs one `bd create` (already `blocked`, metadata included) instead of create, show, and update; a re-record is one update. Attempt lookups use one id-addressed `bd list --id` instead of a metadata-field scan. Startup and periodic recovery look claims up in grouped reads of 25 (closed included) instead of one or two `bd` calls per claim. The persist and recovery time-boxes stretch to a raised `bd_timeout` instead of cutting a slow but healthy call short.
- History lists one row per issue showing its latest outcome with an attempt count (`passed ×3`), in the TUI and in `GET /api/v1/history`; restart-interrupted attempts no longer headline a row, and the TUI detail pane lists the attempts. The state API stays contract v1: `attempt_count` and `attempts` are additive fields on each history row (fixtures and `docs/api/state-api.md` updated).
- Issues closed outside lambe stop as `stopped: closed externally` (was `recovered closed issue`): neutral glyph and colour in the TUI, still listed in history. Older persisted history is mapped on read.
- `lambe status` now fills "last-night gate rejections" from a bounded per-instance log (`<state-dir>/gates/<instance>.jsonl`, last 24h) that the daemon appends on every red gate verdict: issue, attempt, time, and gate output stripped of control characters and truncated to 512 bytes. It prints `(none)` when the log was read and is empty, and `(history unavailable)` only when it cannot be read.
- Heartbeats beat live work before orphaned claims, skip rounds with backoff when the tracker is slow, and start after a small random delay. Startup recovery inspects claims in a bounded parallel, time-boxed pass, so new work dispatches while old claims are still being checked (see `docs/solutions/orphan-claim-heartbeats.md`).
- `lambe status` now exits 2 on a usage error (it was 1) and rejects stray arguments (they were ignored).

### Fixed

- Stale claims no longer starve real work: dozens of orphaned claims used to produce thousands of failing `bd heartbeat` calls and serial multi-minute startup recovery on a single-writer hub.
- Local checkpoints of closed issues are removed instead of being re-read on every restart.
- Dashboard: darker secondary text and error red on a light terminal so they meet WCAG AA contrast on the selected row and status bar.
- `lambe service uninstall` on macOS waits for launchd to finish stopping the daemon, so the agent is gone when the command returns.
- `lambe doctor` says when a fixed `stateapi.addr` port is already in use instead of "daemon not running?".
- `lambe init` no longer fails with `read answer: EOF` when stdin is not a terminal, and it says when it adopts a Beads tracker found above the current directory.

### Tests

- The SIGKILL restart test asserts on every platform that an orphaned agent survives its daemon and is reaped before the re-prompt. The Linux-only failure was the fake agent dying of SIGPIPE, not a Lambe difference.

### Documentation

- `docs/release-template/`: reusable release setup (goreleaser, workflow, scripts) for other Go tools.
- `docs/solutions/dolt-server-mode.md`: running the Beads hub on a loopback `dolt sql-server` (measurements, migration script with a writer fence, rollback, scheduled backup, off-machine backup options).

## [0.10.1] - 2026-11-21

### Added

- Homebrew formula: `brew install andrew-bierman/tap/lambe`.
- `.deb`, `.rpm`, and `.apk` packages (amd64, arm64) with an example systemd unit under `/usr/share/doc/lambe/examples/`.
- Keyless cosign signature for `checksums.txt`, an SBOM per archive, and GitHub build-provenance attestations; `scripts/install.sh` verifies the signature when `cosign` is installed.

### Changed

- Release workflow actions are pinned to commit SHAs.

## [0.10.0] - 2026-11-20

### Added

- `lambe version --json` and a `state API` line in `lambe version`; development builds report `vX.Y.Z-dev+<shortsha>`.
- Opt-in `supervisor.auto_reclaim` / `reclaim_grace` (default off) for multi-host deployments sharing one Dolt server, with `docs/MULTI-MACHINE.md` and isolated multi-host tests.
- Configurable machine/actor identity, `stall_after`, `probe_window`, and `gate.check_timeout`.
- `lambe doctor` preflight checks and `lambe service install|status|uninstall` for launchd/systemd units.
- Optional per-workflow `harness:` (pi, codex, or acp) for named workflows.
- Rex: **Open Run by Workflow…**, **Switch Host…** and **Check Setup** palette actions, live and archived runs with the owning machine, and a headless `contrib/rex/check.lua`.
- Mac app: first-run setup guide (finds running daemons, shows the exact `lambe` commands, starts a daemon in Terminal), live output that follows a running run, Rex handoff on the machine that owns the run (`rex -S`), workflow and machine on runs, run actions in a context menu and the menu bar with keyboard shortcuts, Settings (General, Instances, Tools), and `--demo` with richer sample data.
- Mac app packaging: universal, ad-hoc signed with the hardened runtime, with app-icon checks; Developer ID signing and notarization are a drop-in through `APPLE_*` environment variables. First-launch steps for Gatekeeper are documented.

### Changed

- Mac dashboard: run actions, stall display, and instance views; Rex action wiring.
- Expand `docs/OPERATIONS.md` with the harness, workflow, dashboard, tuning, and restart-semantics reference.
- Redesign the terminal dashboard (`lambe tui`): header with daemon health and counts, Queue / Attention / History tabs, an aligned status table, a preview pane (side by side when wide, stacked or drill-in when narrow), and a contextual key-hint footer; it works down to a 40-column Rex split.
- **License:** lambe is now GPL-3.0-or-later (was MIT). Earlier releases remain under MIT.
- Releases now use a single `vX.Y.Z` tag for the CLI and the Mac app; see `docs/RELEASING.md`.
- `scripts/install.sh` installs the latest release by default (checksum still verified) and prints next steps.

### Fixed

- Rex palette: `lambe.lua` now finds `lambe` through the login shell (Rex's `rex.exec` PATH omits `~/.local/bin`, so **Open Run…** failed), honors `LAMBE_REX_BIN` and `LAMBE_REX_STATE_DIR`, marks a stopped daemon's rows, shows each run's workflow and host, reuses an already-open run window, and adds **Open Run by Workflow…** and **Switch Host…**. `lambe logs -json` reports `Alive` and `Machine`, and Rex detection no longer autostarts a Rex server.
- A tool call that is quiet for longer than `silence_timeout` is no longer killed as a stall (Pi and ACP); `turn_timeout` still bounds it.
- ACP no longer closes its event stream under an in-flight emit (data race, possible send-on-closed-channel panic).
- Sessions recovered after a daemon restart no longer fail every gate with "no gate formulas configured" when no gate snapshot was taken at claim time.
- Mac app: selection no longer survives a change of daemon, so another daemon's issue IDs never show over the current one's data; the daemon version no longer renders as "vv0.9.3".
- Mac app screenshots run in the background without activating the app or raising a window.

## [0.9.2] - 2026-11-20

### Added

- Package the Mac dashboard as a universal (arm64 + x86_64) app ZIP with a checksum.
- Explain Lambe and Rex host support in the docs.

### Changed

- Surface stalled runs in the dashboard and polish narrow layouts, tabs, and run rows.

### Fixed

- Bound stalled Pi runs and Beads subprocesses so a hung child no longer blocks the loop.
- Bundle the mascot icon in the macOS app.

## [0.9.1] - 2026-11-20

### Added

- Open Lambe run output from the Rex palette.

## [0.9.0] - 2026-11-20

### Added

- Add the read-only native macOS dashboard under `apps/dashboard`. It attaches to a daemon through the opt-in (`stateapi.enable`, default off), bearer-authenticated local state API and shows queue, attention, history, and run detail.
- Serve the local HTTP state API on loopback with per-instance endpoint descriptors, queue and history reads, and daemon identity plus per-section timestamps in every response.
- Document the versioned wire contract in `docs/api/state-api.md` and pin it with Go-generated fixtures in `testdata/contract/v1/`.

### Changed

- Mark `POST /api/v1/refresh` as a mutating operator action, so read-only clients stay GET-only.
- Simplify installation docs and point them at the dashboard release.

## [0.8.4] - 2026-11-19

### Added

- Show Beads readiness and agent session details in the dashboard.

## [0.8.3] - 2026-11-19

### Changed

- Show compact run rows, stack the inspector, and give run titles more space in the dashboard.

## [0.8.2] - 2026-11-19

### Added

- Route Beads tasks to named workflows.

### Changed

- Make dashboard views visible in the header.

## [0.8.1] - 2026-11-18

### Changed

- Polish release installation and CLI help; the install and first-run walkthrough now work from a fresh setup.

### Fixed

- Batch Beads attempt lookups during hub recovery.

## [0.8.0] - 2026-11-18

### Added

- Add an Attention view for retries and failed gates, with the latest rejection visible in the inspector.
- Add optional Rex actions for opening the dashboard and demo.
- Expose the current issue ID to shell gates.

### Changed

- Make agent output easier to scan with tool-result markers and fewer repeated tool-start lines.

### Fixed

- Keep dashboard controls visible on narrow terminals and keep filtered selection and follow mode stable across refreshes and resizes.
- Bound optional Rex detection and retry timed-out process-binding reads and writes once.

## [0.1.0] - 2026-11-16

- Publish tagged release archives and checksums; keep local development binaries and state separate from production.
- Explain how to view an existing daemon and that quitting the TUI does not stop it.

- Resolve mise-managed Codex adapters before selecting and checking their Node runtime.

- Complete gate bookkeeping when Beads omits closed records from default reads, and reject changed workspace roots before restoring saved work.

- Allow a separate `workspace.root` so trial and production instances can keep their coding directories isolated.

- Bare CLI commands load the configuration written by init and share the configured tracker’s live-view namespace.

- Show live supervisor phases and agent output with `lambe tui` and `lambe logs`, and use the current Rex status protocol and CLI.
- Select and validate the npm Codex adapter’s Node runtime before claiming work.
- Pin generated workflow paths, provide a native Beads starter with automatic triggering disabled during setup, and show upstream module versions in `lambe version`.
- Re-arm recurring responsibilities after their recorded root closes, decode native Beads status arrays, and persist pour intent to prevent duplicate work after ambiguous failures.


### Fixed

- Restart recovery restores owned attempts, turns, retry deadlines, pinned workflows,
  and gate output from private atomic checkpoints. Surviving agents are replaced
  only after verifying their boot and process identity; unprompted persistence
  failures preserve the attempt, and internal records cannot become agent tasks.

- Polling and reconciliation continue while independent agent workers run, with a workflow-configured concurrency limit and bounded worker shutdown. Standard `agent.max_turns` and `agent.max_concurrent_agents` settings are validated.

- Agent turns now use their configured turn and silence limits instead of inheriting the daemon polling deadline. Formula and gate timeouts remain bounded; timed-out triggers do not create work and timed-out gates retry without consuming an agent attempt.

### Development

- Setup tests isolate platform-specific configuration directories on macOS and
  Linux, and duplicated formula test statements no longer prevent compilation.

### Added

- Codex through the maintained ACP adapter, configurable ACP agent commands,
  harness selection during init, and startup checks for missing executables.
  ACP restores advertised saved sessions and records native backend thread IDs
  before prompting; replayed history is excluded from new turn output.

- Project scaffold: configuration loading with precedence and env flattening
  (`internal/config`), structured logging (`internal/logging`), and mise-pinned
  Go toolchain.
- Beads tracker adapter supporting hub, repo, and in-repo layouts
  (`internal/tracker`).
- Process supervision primitives with a test-only child helper
  (`internal/proc`, `internal/testchild`).
- Symphony core supervisor loop encoding the Symphony specification contract
  (`internal/symphony`, `internal/daemon`).
- Pi agent harness driver (`internal/harness`).
- Verified-completion gates: shell, `gh:run`, and `gh:pr` (`internal/gate`).
- Fault-injection test suite covering kill-mid-reconcile, agent-kill
  resume, limit lockout, and fake-done scenarios.
- MCP server surface (`internal/mcp`): `lambe_spawn`/`status`/`output`/
  `cancel`/`list` over the official go-sdk, with a recursion depth guard.
- ACP harness driver (`internal/harness`): 40+ agents via
  coder/acp-go-sdk, stop-response completion, permission auto-deny.
- Rex terminal surface (`internal/surface/rex`): OSC 7501 status badges
  and labeled sessions with graceful degradation when Rex is absent.
- HTTP state surface (`internal/httpstate`): Symphony §13.7-shaped
  `/api/v1/state`, issue, and refresh endpoints over a typed StateSource.
- Daemon lifecycle: single-instance lock, run loop, and status snapshots,
  exposed via the `lambe` binary (`run`, `status`, `version`).
- Documentation and CI.

[Unreleased]: https://github.com/andrew-bierman/lambe/compare/v0.11.1...HEAD
[0.11.1]: https://github.com/andrew-bierman/lambe/compare/v0.11.0...v0.11.1
[0.11.0]: https://github.com/andrew-bierman/lambe/compare/v0.10.1...v0.11.0
[0.10.1]: https://github.com/andrew-bierman/lambe/compare/v0.10.0...v0.10.1
[0.10.0]: https://github.com/andrew-bierman/lambe/compare/v0.9.2...v0.10.0
[0.9.2]: https://github.com/andrew-bierman/lambe/compare/v0.9.1...v0.9.2
[0.9.1]: https://github.com/andrew-bierman/lambe/compare/v0.9.0...v0.9.1
[0.9.0]: https://github.com/andrew-bierman/lambe/compare/v0.8.4...v0.9.0
[0.8.4]: https://github.com/andrew-bierman/lambe/compare/v0.8.3...v0.8.4
[0.8.3]: https://github.com/andrew-bierman/lambe/compare/v0.8.2...v0.8.3
[0.8.2]: https://github.com/andrew-bierman/lambe/compare/v0.8.1...v0.8.2
[0.8.1]: https://github.com/andrew-bierman/lambe/compare/v0.8.0...v0.8.1
[0.8.0]: https://github.com/andrew-bierman/lambe/compare/v0.7.0...v0.8.0
[0.1.0]: https://github.com/andrew-bierman/lambe/releases/tag/v0.1.0
