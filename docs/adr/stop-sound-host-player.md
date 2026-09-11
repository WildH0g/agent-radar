# Stop sound rides the notify flag through a host-player probe, not the tmux bell

When a `working → stopped` transition sets the `notify` flag — the same flag that already fires the OS notification in `notify_stopped` in `agent-radar-poller` — the plugin also plays a short sound: `afplay` on a system sound on Darwin, else the first available Linux player, preferring a named event sound (`canberra-gtk-play -i complete`) so no audio asset ships with the plugin. Finding no player at all is a silent no-op, and a `@agent-radar-sound` user option (default off — zero surprise on upgrade, the maintainer's preference) gates the cue.

## Why ride the notify flag

The `notify` field emitted by `agent-radar-transition` already *is* the product decision: it is set only when `armed == "1"` — a pane observed `working` that then crossed its idle threshold. First-seen panes arm late and later stop as `unseen-stopped` with `notify = 0`, so they stay silent with zero new logic. Reusing the flag means there is no second definition of "worth hearing about" to drift from the one driving the OS notification.

## Why not the tmux bell

- The bell does have one genuine advantage: it propagates through tmux to the client terminal, so it stays audible over SSH where a host player is not. But tmux has no command to ring it — the only trigger is a pane emitting `\a` — so the plugin would have to spawn a throwaway window per cue and reap it, a race-prone hack for a sound the user's terminal config can silently eat anyway (`visual-bell`, per-terminal bell muting). `notify_stopped` already establishes the probe pattern: `osascript` on Darwin, else `notify-send` if present, else nothing. Sound following the same shape (`afplay`, else a player probe, else nothing) adds no new mechanism class, no new dependency, and no new failure semantics.

## Cost

Sound played on the tmux host is inaudible when driving tmux over SSH — accepted: the operator runs agents on the local machine, and "not at the machine" was explicitly out of scope for this feature. A flurry of stops can otherwise spam sound; a 3-second throttle (at most one sound per window) contains it. Shipping no asset means a stripped-down host with no player and no sound theme silently degrades the cue to the existing OS notification — the same degradation `notify-send`'s absence already causes today, which is why it costs nothing new.
