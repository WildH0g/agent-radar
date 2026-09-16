# OS toast gated by `@agent-radar-notify` around the toast, not by returning from `notify_stopped`

When a `working → stopped` transition sets the `notify` flag, `notify_stopped` in `agent-radar-poller` still fires the OS notification (`osascript` on Darwin, else `notify-send`) — unless `@agent-radar-notify` is not exactly `on`. The option is read with the same `opt` helper as `@agent-radar-sound`, default `on` (today's always-toast, zero surprise on upgrade). The gate wraps only the `osascript` / `notify-send` block. It does not `return` from the function: sound still runs after the toast, gated by its own option.

## Why a second option, not a combined alerts flag

Sound (`stop-sound-host-player.md`) is already a separate opt-in on the same flag. Folding both into `@agent-radar-alerts` would make "quiet the desktop notifier, keep the beep" impossible — the job this option exists for. Two options, two independent `= on` checks, no new mechanism class.

## Why default on, unlike sound

Sound shipped off because it was new. The OS toast already ships; turning it off by default would be a behavior change on upgrade. Unset therefore means `on`, and `[ "$(opt @agent-radar-notify on)" = on ]` is the same comparison sound uses, just with the opposite default. Non-`on` values (`off`, `flase`, `ON`) skip the toast — a typo silences, the same as a sound typo stays quiet.

## Why not `return 0` when notify is off

`notify_stopped` already early-returns when sound is off, *after* the toast. Copying that pattern at the top of the function would skip `afplay` / `canberra-gtk-play` whenever notifications are off, silently coupling the two channels. Wrapping only the OS-toast block keeps the isolation the option promised: status-left, window highlighting, `prefix+a` / `prefix+S`, and sound are unchanged. No new script, no new field on `agent-radar-transition`, no record-store key.

## Rejected alternatives

- **Early `return` from `notify_stopped`.** Kills sound. See above.
- **Gate in `agent-radar-transition` (clear the `notify` flag).** Redefines "worth notifying about" for every future cue that rides that flag. Sound would inherit the silence.
- **Prefix-key toggle or status-left glyph.** A tmux option read at call time already live-applies, same as every other `@agent-radar-*` option. Extra UI is a second surface to document and test.
- **Only exact `off` silences (typos keep toasting).** Safer against misconfig, but a different parser from sound. One comparison shape is enough.

## Cost

A user that wants silence must set the option; the default still floods the host notifier, which is the upgrade-safe choice and the previous behavior. Wrong-case `ON` and typos go silent with no warning — the same failure mode `@agent-radar-sound` already has. Reversible: delete the `if` and the README row.
