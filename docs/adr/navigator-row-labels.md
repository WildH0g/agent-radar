# Navigator row shows the session name, the window name, and the pane's branch, drawn so those fields can be scanned

The navigator popup lists each agent pane as a process name plus `session:window.pane`, sometimes with an age. That string is the jump address. The person who opens it already names the tmux window, and often has several agents in one session, one worktree per window. A `working` agent sits below `unseen-stopped`, so Enter does not reach it, and `fzf` cannot match a name that is not on the row. The session name is glued into the address, so it cannot be read without decoding it.

The pane stays the unit. `CONTEXT.md` says not to treat the window as the unit, and this does not. The target stays `session:window.pane`: what the navigator jumps to, and what `docs/adr/navigator-ordering.md` sorts by. This note does not touch that order. `fzf` stays the navigator.

Each row is the harness, then the session name, then the window name as tmux has it now (including automatic-rename), then `:window.pane`, then that pane's branch. A dim `·` separates those fields. Harness, session, and window name are bold. The address, the branch glyph, and the branch name are dim. A present branch is drawn `⎇ feat/spam-stop`: the branch glyph, a space, then the branch name. No branch, no `git`, and a detached `HEAD` all show `-`, with no glyph. Window index and pane index stay on every row. The process name stays. Age stays, and it is secondary. The three status colors stay, one per state. Age and the branch are dim. Glyphs, font color, and emoji are part of the row so session, window name, and branch can be scanned apart from the address. They are ordinary `fzf` text and may be matched. They must not replace the three status colors. A flat address string is not a row.

The branch is the branch of that agent pane, taken from that pane's directory. The list already has the pane address. It is not the directory of the pane that opened the popup, and it is not a label stored on the window. A failed read must not kill the list. `git` is optional: missing `git` is `-`, not a new required dependency.

## Rejected

- **The address alone.** That is the current row. `fzf` can only match an index, so a `working` agent that is not on top takes the arrow keys.
- **Window name, no branch.** The branch is a typeable reminder of the work on that pane. Dropping it leaves a label that does not say what the pane is on.
- **`detached HEAD` as its own token.** A missing branch, a missing `git`, and a detached `HEAD` are the same absence. One token, `-`.

## Cost

The text people type changes, and the documented navigator line has to change with it. A long window name is cut so the row fits the popup. Keep the tail, not the head: worktree names share a prefix and differ at the end. Do not cut the session name or the window and pane ids. Age and branch give up space first. The shortened text is what `fzf` matches. The cut-off tail need not be typeable.

How often the branch is read is not this decision. A smoke test may move that refresh off the poll tick. That is not a reason to drop the branch, the `⎇` glyph, or the fields.

Revisit when the window name or the branch is no longer something you can type to tell those panes apart.
