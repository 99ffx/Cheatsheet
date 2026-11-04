# Running a Job in the Background (nohup)

[← Back to index](README.md)

Use this to start a job on the remote machine so it **keeps running after I log out or turn off my laptop**.

## Template

```bash
cd ~/PhD2026/Run
mkdir -p logs

TRAINER=<TRAINER_NAME> nohup bash scripts/train_local.sh <SITE> <FOLD> \
    > logs/<RUN_NAME>.out 2>&1 &
echo $! > logs/<RUN_NAME>.pid      # save the PID so I don't lose it
```

| Part | Meaning |
|---|---|
| `TRAINER=...` | Environment variable that selects the nnU-Net trainer, e.g. `nnUNetTrainer_250epochs` (see [nnunet.md](nnunet.md)) |
| `nohup` | Keeps the process alive when the SSH session closes |
| `bash scripts/train_local.sh <SITE> <FOLD>` | The training script, with the site ID and the fold number |
| `> logs/<RUN_NAME>.out` | Writes normal output to a log file |
| `2>&1` | Sends errors to the same log file |
| `&` | Runs the job in the background and returns the prompt |
| `echo $! > ...pid` | `$!` is the PID of the last background job; saving it means I can find it later |

## Example

```bash
TRAINER=nnUNetTrainer_250epochs nohup bash scripts/train_local.sh 104 0 \
    > logs/run_site04_fold0.out 2>&1 &
echo $! > logs/run_site04_fold0.pid
```

The terminal prints something like `[1] 2356346`. The second number is the **PID**.
Check on it with the commands in [process-status.md](process-status.md).

## Several folds, one after another

```bash
TRAINER=nnUNetTrainer_250epochs nohup bash -c '
  for FOLD in 0 1 2 3 4; do
    bash scripts/train_local.sh 104 $FOLD > logs/run_site04_fold${FOLD}.out 2>&1
  done
' > logs/run_site04_all.out 2>&1 &
echo $! > logs/run_site04_all.pid
```

## Two jobs at once on different GPUs

```bash
CUDA_VISIBLE_DEVICES=0 TRAINER=nnUNetTrainer_250epochs nohup bash scripts/train_local.sh 104 0 \
    > logs/run_site04_fold0.out 2>&1 &
echo $! > logs/run_site04_fold0.pid

CUDA_VISIBLE_DEVICES=1 TRAINER=nnUNetTrainer_250epochs nohup bash scripts/train_local.sh 104 1 \
    > logs/run_site04_fold1.out 2>&1 &
echo $! > logs/run_site04_fold1.pid
```

## Start it from my laptop in one line

```bash
ssh <user>@<host> 'cd ~/PhD2026/Run && TRAINER=nnUNetTrainer_250epochs nohup bash scripts/train_local.sh 104 0 > logs/run_site04_fold0.out 2>&1 & echo $!'
```

## Stronger detachment: setsid

If a job still dies when I log out, use `setsid` (starts it in a new session):

```bash
TRAINER=nnUNetTrainer_250epochs setsid bash scripts/train_local.sh 104 0 \
    > logs/run_site04_fold0.out 2>&1 < /dev/null &
```

## nohup vs tmux vs SLURM

| Tool | Use when |
|---|---|
| `nohup` | Fire-and-forget on a personal or lab workstation |
| [tmux](tmux.md) | I want to reattach and see the live terminal |
| [SLURM](slurm.md) | I'm on an HPC cluster. Never train on a login node |
