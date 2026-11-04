# GPU

[← Back to index](README.md)

## Check GPU use

```bash
nvidia-smi                     # snapshot: memory, utilization, which PIDs use each GPU
watch -n 2 nvidia-smi          # refresh every 2 seconds (Ctrl+C to exit)
```

Compact view:

```bash
nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total --format=csv
```

Which processes are on the GPU:

```bash
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
```

## Pick which GPU a job uses

```bash
CUDA_VISIBLE_DEVICES=1 TRAINER=nnUNetTrainer_250epochs nohup bash scripts/train_local.sh 104 0 \
    > logs/run_site04_fold0.out 2>&1 &
```

Inside the job, the chosen GPU shows up as `cuda:0`.

## Check that PyTorch sees the GPU

```bash
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.device_count())"
```

## Signs of trouble

| What I see | Likely cause |
|---|---|
| `CUDA out of memory` in the log | Another job is on the same GPU, or the patch/batch size is too big |
| GPU utilization ~0% but the job is running | Data loading is the bottleneck, or the job is still preprocessing |
| GPU memory still used after `kill` | A python child is still alive; see [stopping-jobs.md](stopping-jobs.md) |
