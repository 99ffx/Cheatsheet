# Common Pitfalls

[← Back to index](README.md)

| Problem | Cause | Fix |
|---|---|---|
| `ps -p <PID>` finds nothing, but the job should be running | I'm logged in to a different node | Check `hostname`, then `ssh` to the right node ([ssh.md](ssh.md)) |
| `conda: command not found` or wrong python in the log | Detached shells don't load my interactive environment | Activate conda **inside** the script (see below) |
| nnU-Net complains about missing paths | `nnUNet_raw` / `nnUNet_preprocessed` / `nnUNet_results` not set | Export them inside the script ([nnunet.md](nnunet.md)) |
| `No such file or directory: logs/...` | `logs/` folder doesn't exist | `mkdir -p logs` before starting |
| Job died when I closed the laptop | Started without `nohup`, or forgot the `&` | Use the template in [background-jobs.md](background-jobs.md), or tmux |
| `CUDA out of memory` | Another job on the same GPU | `nvidia-smi`, then set `CUDA_VISIBLE_DEVICES` ([gpu.md](gpu.md)) |
| GPU still busy after `kill` | Python children survived | `pkill -P <PID>` ([stopping-jobs.md](stopping-jobs.md)) |
| All jobs gone the next morning | Server rebooted | Only SLURM requeueing or a systemd service survives a reboot. Resume nnU-Net with `--c` |
| Training job killed on the cluster login node | Heavy compute isn't allowed there | Use `sbatch` ([slurm.md](slurm.md)) |
| Job crashes writing checkpoints | Disk full or over quota | `df -h ~`, `du -sh $nnUNet_results/*` |

## Conda inside a script

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate <ENV>
```

(Adjust the path if conda is installed somewhere else: `conda info --base` shows it.)

## Disk usage

```bash
df -h ~                          # free space on my home filesystem
du -sh $nnUNet_results/*         # size of each results folder
du -sh ~/* | sort -h | tail      # biggest folders in home
```
