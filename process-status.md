# Checking Job Status (PID)

[← Back to index](README.md)

## Is it still running?

```bash
ps -p <PID> -o pid,etime,stat,cmd
```

- Prints one line → **alive**
- Prints only the header → **finished or crashed** (check the end of the log, see [logs.md](logs.md))

Using the saved PID file:

```bash
ps -p $(cat logs/run_site04_fold0.pid) -o pid,etime,stat,cmd
```

| Column | Meaning |
|---|---|
| `pid` | Process ID |
| `etime` | Elapsed time, `[days-]hh:mm:ss` |
| `stat` | `R` = running, `S` = sleeping (normal while waiting on the GPU or disk), `D` = waiting on disk, `Z` = zombie, `T` = stopped |
| `cmd` | The command that started it |

## Check every saved PID at once

```bash
for f in logs/*.pid; do
  pid=$(cat "$f")
  if ps -p "$pid" > /dev/null; then echo "RUNNING  $f ($pid)"; else echo "DONE     $f ($pid)"; fi
done
```

## See child processes

nnU-Net starts python children under my bash script:

```bash
pstree -p <PID>
```

## Find my jobs without the PID

```bash
ps -ef | grep nnUNet | grep -v grep          # all nnU-Net processes
ps -u $USER -o pid,etime,stat,cmd            # everything I'm running
pgrep -af train_local.sh                     # PID + command for my training script
```

## Interactive overview

```bash
top -u $USER       # q to quit
htop -u $USER      # nicer, if installed
```

> `jobs` and `fg` only work in the **same terminal** that started the job.
> After logging out and back in, use `ps`.
