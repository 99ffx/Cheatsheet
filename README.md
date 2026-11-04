# Command-Line Cheatsheet

Commands I use for running and monitoring nnU-Net training jobs on remote machines.
Each topic has its own file.

| # | File | What's in it |
|---|---|---|
| 1 | [ssh.md](ssh.md) | Connecting to the remote machine, checking the host, copying files |
| 2 | [background-jobs.md](background-jobs.md) | Running a job with `nohup` so it survives logout / laptop off |
| 3 | [process-status.md](process-status.md) | Checking whether a job (PID) is still running |
| 4 | [logs.md](logs.md) | Watching and searching log files |
| 5 | [gpu.md](gpu.md) | Checking GPU use, picking a GPU |
| 6 | [stopping-jobs.md](stopping-jobs.md) | Killing jobs safely |
| 7 | [tmux.md](tmux.md) | Persistent terminal sessions |
| 8 | [slurm.md](slurm.md) | SLURM jobs on HPC clusters (e.g. Berzelius) |
| 9 | [nnunet.md](nnunet.md) | nnU-Net environment, trainers, training, results |
| 10 | [pitfalls.md](pitfalls.md) | Common mistakes and how to fix them |

## the commands I use most (Update late OCT26)

```bash
# Start training in the background and save the PID
TRAINER=nnUNetTrainer_250epochs nohup bash scripts/train_local.sh 104 0 \
    > logs/run_site04_fold0.out 2>&1 &
echo $! > logs/run_site04_fold0.pid

# Is it still running?
ps -p $(cat logs/run_site04_fold0.pid) -o pid,etime,stat,cmd

# Watch the output
tail -f logs/run_site04_fold0.out

# GPU use
nvidia-smi
```
