# Poller self-heals from the status heartbeat, not a new supervisor

`agent-radar-status` now re-invokes `agent-radar-poller start` (detached, via
`run-shell -b`) on every render, instead of the poller only ever being started
once from `agent-radar.tmux` when tmux sources its config.

## Why this looks redundant

`start()` already exits immediately in the common case (the `kill -0` guard:
no-op if a live pid owns `@agent-radar-pid`), so calling it on a heartbeat that
may fire every few seconds looks like pointless churn.

## Why it's needed anyway

The 27 Jul fix (`628617e`) hardened every *graceful* way the daemon could stop
tracking: a bad poll cycle under `set -e`, and an alive-check that needed an
attached client. It added `trap ... EXIT` so `@agent-radar-pid` "never goes
stale on unexpected exit" — but a trap can only run on signals a process can
catch. A SIGKILL (OOM killer, `kill -9`, a forceful process-group teardown)
leaves `@agent-radar-pid` pointing at a pid that no longer exists, and nothing
was left to notice: the initial `run-shell -b` in `agent-radar.tmux` fires once,
at config load, not on a cadence. Observed on a real machine: the daemon was
dead for 5+ hours with a stale pid still recorded, silently freezing every
downstream signal (notifications, status segment, window highlight) while
looking fine from the outside.

## Why the status heartbeat, not a dedicated mechanism

- A separate supervisor (cron/launchd/systemd) breaks "self-contained tmux
  plugin" — the one thing this project explicitly is.
- A dedicated tmux hook (`session-created`, `client-attached`, ...) only fires
  on those events, not on a cadence, so a long-lived session could stay broken
  indefinitely between them.
- `agent-radar-status` is already the plugin's only recurring tick (tmux drives
  it every `status-interval`), already used to drive `sync_window_flags`.
  Piggybacking keeps the liveness check in exactly one place (`start()`
  itself) instead of duplicating it.

## Cost

Recovery latency is bounded by `status-interval` (tmux default 15s), not
instant. Users who set `status off` already lose window highlighting (it's
driven by the same script) and now also lose self-heal — an existing coupling,
not a new one.

`start()`'s dead-pid takeover is read-then-write across two separate tmux
round trips, not atomic — a fast heartbeat can in principle fire two takeovers
back-to-back before the first write is visible to the second, leaving two
loops alive. (A live-seeming duplicate was chased on a `status-interval 1`
machine; `ps` kept showing a second `agent-radar-poller start` line, but its
ppid was the *real* daemon, not the server — `poll_once`'s own pipes and
`$(command substitutions)` fork subshells that never exec, so they inherit
and display the parent's argv. Not a second daemon. Left as a reminder: check
ppid before trusting `ps` command text on a duplicate-looking process.) The
read-then-write gap is still real even though it wasn't caught in the act, so
the while loop re-checks ownership every cycle and `break`s if someone else's
write has since won — a loser steps aside within one poll interval instead of
running forever alongside a winner, no locking needed.

Separately, forking `run-shell -b` unconditionally on every render is real,
measured cost regardless of the race: on a machine with 25 real panes a poll
cycle takes ~3.4s, and a fresh `sh` + tmux round trip every second adds up.
The liveness check (`show-option` + `kill -0`) now runs in `agent-radar-status`
itself first; `run-shell -b` (and the `sh` startup behind it) only happens
when that check says the daemon is actually dead.
