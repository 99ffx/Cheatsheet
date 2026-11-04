# tmux: Persistent Terminal Sessions

[← Back to index](README.md)

Better than `nohup` when I want to come back and see the live terminal.

## Basic use

```bash
tmux new -s <NAME>             # start a new session
# ... run commands normally ...
# Ctrl+b then d                # detach (the job keeps running)

tmux ls                        # list sessions
tmux attach -t <NAME>          # reattach
tmux kill-session -t <NAME>    # close the session and stop what's inside it
```

## Example

```bash
tmux new -s site04
cd ~/PhD2026/Run
TRAINER=nnUNetTrainer_250epochs bash scripts/train_local.sh 104 0 2>&1 | tee logs/run_site04_fold0.out
# Ctrl+b then d
```

`tee` shows the output on screen and saves it to the log at the same time.

## Keys inside tmux

Press `Ctrl+b` first, then:

| Key | Action |
|---|---|
| `d` | Detach |
| `c` | New window |
| `n` / `p` | Next / previous window |
| `0`–`9` | Go to window number |
| `%` / `"` | Split vertically / horizontally |
| arrow keys | Move between split panes |
| `[` | Scroll mode (arrows / PgUp, `q` to exit) |
| `$` | Rename session |

## If tmux isn't installed: screen

```bash
screen -S <NAME>     # start
# Ctrl+a then d      # detach
screen -ls           # list
screen -r <NAME>     # reattach
```
