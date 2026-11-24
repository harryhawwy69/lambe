<div align="center">

<img src="docs/assets/lambe-mascot.png" alt="Lambe, a friendly lambeosaurus" width="140">

# Lambe

**Keep your coding agents working. Know when the work is actually done.**

[![CI](https://github.com/andrew-bierman/lambe/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/andrew-bierman/lambe/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/andrew-bierman/lambe)](https://github.com/andrew-bierman/lambe/releases/latest)
[![Go Reference](https://pkg.go.dev/badge/github.com/andrew-bierman/lambe.svg)](https://pkg.go.dev/github.com/andrew-bierman/lambe)
[![Go Report Card](https://goreportcard.com/badge/github.com/andrew-bierman/lambe)](https://goreportcard.com/report/github.com/andrew-bierman/lambe)
[![License: GPL-3.0-or-later](https://img.shields.io/badge/license-GPL--3.0--or--later-blue.svg)](LICENSE)

</div>

Lambe is a supervisor for coding agents. It takes work from your
[Beads](https://github.com/gastownhall/beads) queue, runs an agent on each
task, restarts the ones that stall, and marks a task done only when your
checks pass.

<p align="center">
  <img src="docs/assets/dashboard.png" alt="The Lambe terminal dashboard in a Rex window, showing a queue of runs with live agent output" width="900">
</p>

## Why Lambe

- **Done means verified.** A task finishes when its completion gate passes
  (a shell command, a GitHub Actions run, or a pull request check), not when
  the agent process exits.
- **Stalls and crashes recover.** Lambe notices a silent agent, retries with
  backoff, and resumes from a checkpoint after a restart. After the attempt
  limit, the task goes to a human with the reason attached.
- **Bring your own tools.** Beads for the queue, [Pi](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent),
  Codex or any [ACP](https://agentclientprotocol.com) agent for the work,
  [Rex](contrib/rex/README.md) or any terminal for watching.

## Quickstart

```sh
curl -fsSL https://raw.githubusercontent.com/andrew-bierman/lambe/master/scripts/install.sh | sh
lambe init     # from the project that holds your Beads tracker
lambe run      # start the supervisor
```

Before `lambe run` you need [`bd`](https://github.com/gastownhall/beads) with a
tracker (`bd init`), an agent (Pi by default), and a `[gate]` completion check
in the formula `lambe init` writes. `lambe doctor` lists whatever is missing.

Other ways to install:

```sh
brew install andrew-bierman/tap/lambe        # macOS and Linux (also installs beads)
go install github.com/andrew-bierman/lambe/cmd/lambe@latest
```

Each [release](https://github.com/andrew-bierman/lambe/releases) also has
`.deb`, `.rpm`, and `.apk` packages for amd64 and arm64. They install
`/usr/bin/lambe` and an example unit under `/usr/share/doc/lambe/examples/`;
no service is enabled. Every release artifact (archives, deb/rpm/apk, SBOMs,
`checksums.txt`) is signed with cosign and carries a build attestation.

### Verifying a release

Download an artifact and its `.sigstore.json` bundle, then:

```sh
cosign verify-blob lambe_X.Y.Z_linux_amd64.deb \
  --bundle lambe_X.Y.Z_linux_amd64.deb.sigstore.json \
  --certificate-identity-regexp '^https://github.com/andrew-bierman/lambe/\.github/workflows/release\.yml@refs/tags/v' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
gh attestation verify lambe_X.Y.Z_linux_amd64.deb --repo andrew-bierman/lambe
```

Same for the `.tar.gz` archives. Details and the checksum flow are in
[docs/RELEASING.md](docs/RELEASING.md#verifying-a-release).

The installer verifies the archive against the release checksum (and the
cosign signature when `cosign` is on your PATH) and puts
`lambe` in `~/.local/bin`. Set `LAMBE_VERSION` to pin a release. Prefer to
read it first? It is [scripts/install.sh](scripts/install.sh), and archives
are on the [releases page](https://github.com/andrew-bierman/lambe/releases).

Want to look around before setting anything up?

```sh
lambe tui -demo
```

The demo runs in memory with sample runs. No tracker, agent, or daemon needed.

## Tour

### Terminal dashboard

`lambe tui` attaches to a running daemon (quitting it leaves the daemon
running). Queue shows ready, active, and blocked work with live agent output.
Press `:` for the command palette, `1` `2` `3` to switch views, `/` to filter.

<p align="center">
  <img src="docs/assets/stalled.png" alt="A stalled run selected in the Queue view" width="900">
</p>

Attention gathers everything that needs you: retries, failed gates, and
blocked tasks with the reason and dependency IDs. History keeps recent
attempts and their outcomes.

<p align="center">
  <img src="docs/assets/attention.png" alt="The Attention view with a retrying, an escalated, and a blocked task" width="440">
  <img src="docs/assets/history.png" alt="The History view with recent attempts" width="440">
</p>

### Narrow terminals and split panes

The layout adapts. In a 40-column split it shows one pane at a time and fits
more runs per page, so the dashboard can sit beside your shell.

<p align="center">
  <img src="docs/assets/dashboard-mobile.png" alt="The dashboard in a narrow Rex split next to a shell pane" width="900">
</p>

### Rex

Rex users can open the dashboard or the demo from the
command palette, search live and archived runs, and show daemon status as a
badge. See [Lambe in Rex](contrib/rex/README.md).

### macOS app

`apps/dashboard` is a native SwiftUI app that attaches to a running daemon
over its local, token-guarded state API and shows the same Queue, Attention,
and History views, live run output, and stall warnings. It is read-only, and
hands a run to Rex (on the machine running the agent) or Terminal. Its run controls are read-only by design until the state API gets mutating actions. A setup
guide finds your daemon or shows the commands to start one, and a demo mode
needs no daemon. Build instructions, first-launch steps for the ad-hoc-signed
download, and shortcuts are in [apps/dashboard/README.md](apps/dashboard/README.md).

<p align="center">
  <img src="docs/assets/dashboard-app-light.png" alt="The LambeDash Mac app showing a queue with one stalled run and live agent output" width="900">
</p>

## Supported agents

| Agent | How | Notes |
| --- | --- | --- |
| [Pi](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent) | default | Reopens its saved workspace session after a restart |
| Codex | `lambe init -harness codex` | Through the [Codex ACP adapter](https://github.com/agentclientprotocol/codex-acp) |
| Other ACP agents | `harness: acp` | Named agent or explicit argv, for example OpenCode |

Setup details are in [Agent harnesses](docs/REFERENCE.md#agent-harnesses).

## Completion gates

Gates live next to your tasks in `.beads/formulas/` and run when the agent
says it is finished. Three kinds ship today: `shell`, `gh:run`, and `gh:pr`.
A rejected gate counts as a failed attempt and the task is retried.
[Run a disposable task](docs/OPERATIONS.md#run-a-disposable-task) walks
through a throwaway tracker, one task, and a shell gate.

## More than one machine

Lambe claims work through Beads, so two supervisors can share a tracker
without taking the same task.

## FAQ

**Does Lambe write code?**
No. It runs the agent you configure and supervises it. Lambe decides when to
retry, when to stop, and whether the result passed your checks.

**What happens if the daemon or my machine restarts?**
Lambe keeps checkpoints in its state directory and resumes pinned workflows,
attempts, and retry deadlines. See
[Keeping the daemon running](docs/REFERENCE.md#keeping-the-daemon-running)
for launchd and systemd units.

**Will agents touch my other tasks?**
Only tasks that match your configured `labels`. On a shared tracker, set
labels so Lambe claims just its own lane.

**What do agents have access to?**
Whatever the `lambe` process has: your environment, filesystem, and
credentials. See [SECURITY.md](.github/SECURITY.md).

**Is there a stable API?**
An opt-in local state API serves JSON on loopback behind a per-start bearer
token. The contract is in [docs/api/state-api.md](docs/api/state-api.md).
Scripts that do not want the API can run `lambe status -json` (the same
snapshot shape, read from the tracker) and `lambe state-dir -json` (where
the state directory and descriptors are).

## How it relates to other projects

- **[Symphony](https://github.com/openai/symphony)** is the specification
  Lambe implements, pinned to commit `8001b52`. Lambe adds durable state
  and claim fencing on top and is not affiliated with OpenAI.
- **[Beads](https://github.com/gastownhall/beads)** is the task tracker and
  the source of truth for what is queued and claimed.
- **[Rex](contrib/rex/README.md)** is a terminal multiplexer Lambe integrates
  with. It is optional.

## Documentation

| | |
| --- | --- |
| [Operations guide](docs/OPERATIONS.md) | Run a disposable task, day-to-day operation |
| [Reference](docs/REFERENCE.md) | CLI exit codes and completions, harnesses, workflows, restart behavior |
| [Configuration reference](docs/reference/config.md) | Every config key, default and environment variable (generated) |
| [Troubleshooting](docs/TROUBLESHOOTING.md) | Symptoms, causes and fixes; start with `lambe doctor` |
| [Glossary](docs/GLOSSARY.md) | Symphony, Beads, gates, attempts, claims, leases, workflows |
| [Upgrading](docs/UPGRADING.md) | What to do on each upgrade and what changed |
| [Architecture](docs/ARCHITECTURE.md) | Contributor overview: package map and design decisions |
| [State API](docs/api/state-api.md) | The versioned local JSON contract |
| [Multiple machines](docs/MULTI-MACHINE.md) | Sharing one tracker across hosts |
| [Releasing](docs/RELEASING.md) | How releases are cut and versioned |
| [Changelog](CHANGELOG.md) | Release notes |

## Contributing

Issues and pull requests are welcome. Start with
[CONTRIBUTING.md](CONTRIBUTING.md). Report security problems privately, as
described in [SECURITY.md](.github/SECURITY.md).

## License

Lambe is licensed under the [GNU General Public License v3.0 or
later](LICENSE). The Symphony specification (Apache-2.0) and Beads (MIT) are
attributed in [NOTICE](NOTICE).
