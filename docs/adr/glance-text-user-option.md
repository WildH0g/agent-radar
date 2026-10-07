# Glance text is a poller-written user option, not a status command

tmux expands `status-format` on `status-interval` and does not handle keys while those `#()` jobs run. A job that calls `tmux` waits on the same server ([tmux#1854](https://github.com/tmux/tmux/issues/1854)). The glance is installed as that job. From a normal shell it took 0.38–0.40 s, and it calls `tmux show-option`, `display-message`, `list-windows`, and `set-window-option` while the server is held. `refresh-client -S` took 0.21–0.26 s locally and 0.50–0.61 s on a VPS with that job installed. Replacing the job with a file read dropped those refreshes to about 0.10 s and 0.20 s. The lock is the status job, not the poller. The poller is a normal client and waits its turn.

The poller writes the rendered glance into the user option `@agent-radar-glance-text`. The status format begins with `#{@agent-radar-glance-text}`, not `#{E:@agent-radar-glance-text}`. The only command in that slot is a no-output heartbeat `#()`. A healthy refresh does not run `agent-radar-glance` and does not call `tmux`. On tmux 3.4 a plain `#{@option}` applies `#[` styles and leaves `%`, commas, and `#{session_name}` literal. `#{E:}` expands formats inside the value, so a window name could inject. That form is forbidden. A client-keyed cache file is forbidden. `docs/adr/second-status-line.md` already made the line server-global.

A dead poller cannot restart itself. `docs/adr/poller-self-heal.md` records a SIGKILL that left a stale pid for five hours because nothing on a cadence noticed. Today that cadence is the glance job. Removing every `tmux` call from the status format would reopen that failure. A no-output heartbeat may call `tmux run-shell -b` to start the poller only when the poller's pid file is dead. While that pid is alive, the heartbeat must not call `tmux` and must not print into the line. Reading `@agent-radar-pid` from the status job is itself a `tmux` call, so the heartbeat reads the pid file.

- **Do nothing.** The keystroke lock stays, and it grows with the number of agent panes.
- **Keep `agent-radar-glance` as the `#()` and only drop its nested `tmux` calls.** The server is held for the whole job. The measured 0.38–0.40 s is the lock, not a side trip.
- **`#(cat <file>)`.** That proved the class of fix. It still forks once per client per refresh, can paint one refresh late, and needs a path that does not collide across `tmux -L` servers. A user option is one `set-option` from the poller and no shell on the healthy path.
- **A second supervisor.** Already refused in `docs/adr/poller-self-heal.md`.
- **Forbid the dead-path `run-shell -b` as well.** That is the five-hour stale pid again.

This does not supersede `docs/adr/second-status-line.md`. That note still owns the row, the stock-text overwrite, and the untouched first line. It retires one claim there: with the glance on, its render no longer keeps the window highlight and the seen mark on the `status-interval`. Those two must not move back into a status `#()` that calls `tmux`. How they are kept is not this decision.

This note does not decide the navigator, the CPU pills on the first status line, `capture-pane`, or the seen-mark algorithm. A quick refresh is not evidence if some other `#()` is still slow. Do not force `refresh-client -S` after each write. That re-runs every client's status jobs. The line can be up to one poll interval staler than a live `#()` compute. The screen still updates on `status-interval`.

## How to tell it worked

- `tmux show -gqv` of the glance slot begins with `#{@agent-radar-glance-text}` and does not name `agent-radar-glance` or `tmux`. The only `#()` is the no-output heartbeat.
- While the poller pid is alive, a status refresh spawns neither `tmux` nor `agent-radar-glance`.
- `refresh-client -S` lands near the file-read floor, about 0.10 s locally and 0.20 s on the VPS, and does not grow with agent count the way the 0.38–0.40 s glance job did.
- An attached client shows the tinted row, not the literal `#[fg=` text. A `#{session_name}` stored in the option stays literal.
- Kill the poller and wait one `status-interval`. A new pid must appear. That is the dead path, not the render.
- Do not score the navigator, the CPU pills, or `capture-pane`.

Reopen this if tmux stops holding the server during a status `#()`, if `#{@option}` stops applying `#[` styles, or if a plain `#{@option}` starts expanding `#{…}` the way `#{E:}` does. Wanting the glance script back in the status slot is not a trigger.
