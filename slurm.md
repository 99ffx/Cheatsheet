# SLURM (HPC Clusters, e.g. Berzelius)

[← Back to index](README.md)

On a cluster, **don't run training on the login node with nohup**. Submit it to the scheduler.

## Submit and monitor

```bash
sbatch scripts/train.sbatch 104 0      # submit; prints the job ID
squeue -u $USER                        # my queued and running jobs
squeue -u $USER --start                # estimated start time for pending jobs
scontrol show job <JOBID>              # full details of one job
tail -f logs/<JOBNAME>_<JOBID>.out     # follow the output
```

## After the job ends

```bash
sacct -j <JOBID> --format=JobID,JobName,State,Elapsed,ExitCode,MaxRSS
sacct -u $USER --starttime=today       # everything I ran today
```

| State | Meaning |
|---|---|
| `PENDING` (`PD`) | Waiting in the queue |
| `RUNNING` (`R`) | Running |
| `COMPLETED` | Finished with exit code 0 |
| `FAILED` | Script exited with an error, check the log |
| `TIMEOUT` | Hit the `--time` limit |
| `OUT_OF_MEMORY` | Ran out of RAM |
| `CANCELLED` | Cancelled by me or an admin |

## Cancel

```bash
scancel <JOBID>
scancel -u $USER                       # cancel ALL my jobs (careful)
```

## Minimal sbatch script

`scripts/train.sbatch`:

```bash
#!/bin/bash
#SBATCH --job-name=nnunet_site104
#SBATCH --gpus=1
#SBATCH --time=24:00:00
#SBATCH --output=logs/%x_%j.out        # %x = job name, %j = job ID

source ~/miniconda3/etc/profile.d/conda.sh
conda activate <ENV>

TRAINER=nnUNetTrainer_250epochs bash scripts/train_local.sh "$1" "$2"
```

## Submit all folds

```bash
for FOLD in 0 1 2 3 4; do
  sbatch scripts/train.sbatch 104 $FOLD
done
```

## Interactive GPU session (for debugging)

```bash
salloc --gpus=1 --time=01:00:00
```

> Cluster-specific rules (accounts, partitions, GPU types) differ. Check the cluster's own documentation.
