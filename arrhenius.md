# Running FednnUNet on Arrhenius

How to run this project's federated nnU-Net (FednnUNet) training on the NAISS Arrhenius GPU partition, from choosing the environment to reading the results. Written 2026-11-13; module names and versions change, so check `module avail` on the system if a name is not found.

## 1. How Arrhenius is laid out

| Part | What it is | What it means for us |
|---|---|---|
| Login node (`login.hpc.arrhenius.naiss.se`) | x86_64 (AMD EPYC) | Edit files, copy data, submit jobs. Cannot run anything built for the GPU nodes. |
| GPU partition (`-p gpu`) | 4 x NVIDIA GH200 per node, **ARM (aarch64)** CPUs | Builds the env and runs the training. |
| `GPU/...` modules | only load on GPU nodes | Python for the env must be loaded inside a GPU job. |
| `/home/<user>` | 30 GiB, 500,000 files, private | Code only. |
| `/nobackup/proj/disk/<project>` | project storage, shared | Env, datasets, results. |

Because the login node and the GPU nodes have different processors, **the env must be built and used on a GPU node**. Running the env's Python on the login node gives `Exec format error`.

Our layout:

| What | Where |
|---|---|
| Code (this repo, incl. `nnUNet/` and `FednnUNet/`) | `/home/skitti/code/Baseline_nnUNet_BraTS` |
| Python venv | `/nobackup/proj/disk/fedmed/shared/envs/fednnunet` |
| `nnUNet_raw`, `nnUNet_preprocessed` | `/nobackup/proj/disk/fedmed/shared/BraTS21/` |
| FL run outputs (`FL_RUNS_ROOT`) | `/nobackup/proj/disk/fedmed/shared/BraTS21/fl_runs` |
| Slurm logs | `~/Run/slurm_logs/` |

## 2. Choosing the environment

**Chosen: the `GPU/Python` "bare" module + a venv + pip-installed torch 2.14.0 (CUDA 13).**

| Option | Verdict | Why |
|---|---|---|
| `GPU/Python/3.13.5-bare-gcc-2025b-eb` + venv + pip | **Used** | NSC-built Python for the ARM GPU nodes; every package comes from pip and is recorded in one requirements file; few files; the route NAISS documents. |
| `PyTorch/2.9.1-CUDA-12.9.1` module | Not usable | nnU-Net's `pyproject.toml` requires `torch>=2.1.2,!=2.9.*`, so pip would replace the module's torch anyway. |
| `GPU/Python/...-bundle-...` module | Not needed | Its NumPy/SciPy/pandas/mpi4py would either be ignored by the venv or mix with pip's versions. We do not use MPI. |
| `GPU/buildenv-nvhpc` / `GPU/buildenv-gcccuda` | Not needed | Compilers and CUDA dev tools for building code; no Python. All our packages are ready-made ARM wheels, and torch ships its own CUDA. NAISS: do not load a buildenv in a normal batch job. Only load one for a single `pip install` that must compile. |
| `Miniforge` module (conda) | Not used | Exists only for the CPU partition (no `GPU/Miniforge`), so envs it makes are x86. NAISS discourages conda (many files, slow start-up). |

Why torch 2.14.0: it is the version on the other machines (lnx00293, Harley), so FL results on Arrhenius are comparable with the CL and local runs. Python is 3.13 here instead of 3.12; nnU-Net and Flower support both.

Checked before setup: torch 2.14.0+cu130, flwr 1.39.0 (with grpcio), and all of nnU-Net's dependencies exist as ready-made `aarch64` / Python 3.13 wheels. Only `medpy` is source-only (used by `src/analysis/test_evaluation.py`, not by training).

## 3. Build the environment (once, on a GPU node)

Open an interactive session on a GPU node (needed: the env is ARM and `GPU/` modules only load there):
```bash
interactive -A naiss2026-3-793-gpu -p gpu --gpus 1 -t 01:30:00
```

Inside the session:

```bash
uname -m                     # aarch64
nvidia-smi                   # one GH200
cd /home/skitti/code/Baseline_nnUNet_BraTS
```

Load Python (check the exact name with `module avail GPU/Python`):
```bash
module --quiet purge
module load GPU/Python/3.13.5-bare-gcc-2025b-eb
python --version             # Python 3.13.5
```

Create and activate the venv on project storage:
```bash
python -m venv /nobackup/proj/disk/fedmed/shared/envs/fednnunet
source /nobackup/proj/disk/fedmed/shared/envs/fednnunet/bin/activate
python -m pip install --upgrade pip
```

Install torch, then the project (from the repo root):
```bash
python -m pip install torch==2.14.0 --index-url https://download.pytorch.org/whl/cu130
python -m pip install -e nnUNet
python -m pip install "flwr==1.39.0"
python -m pip install --no-deps -e FednnUNet
python -m pip install openpyxl medpy
```

- `python -m pip` always installs into the venv of the `python` you run.
- `-e nnUNet` installs the repo's own nnU-Net copy (the exact commit), not whatever is newest on PyPI. `-e` writes a link in the venv to `/home/.../nnUNet`, so do not move the repo folder (or rerun the two `-e` installs after moving it).
- `--no-deps` stops FednnUNet from pulling its own torch / nnU-Net / Flower.
- If `medpy` fails to build, skip it until evaluation, or load a buildenv just for that install: `module load GPU/buildenv-gcccuda/2026.03-cu13.0`.

Check, record, leave:
```bash
python -c "import torch, flwr, nnunetv2, fednnunet; print('torch', torch.__version__, '| cuda', torch.cuda.is_available(), torch.cuda.get_device_name(0), '| flwr', flwr.__version__)"
python check_env.py          # see Appendix; must end with ALL OK
python -m pip freeze > /nobackup/proj/disk/fedmed/shared/envs/fednnunet_requirements.txt
exit
```
Expected: `torch 2.14.0+cu130 | cuda True NVIDIA GH200 ... | flwr 1.39.0`.

Adding a package later: open an interactive GPU session, then
```bash
module load GPU/Python/3.13.5-bare-gcc-2025b-eb
source /nobackup/proj/disk/fedmed/shared/envs/fednnunet/bin/activate
python -m pip install <package>
python -m pip freeze > /nobackup/proj/disk/fedmed/shared/envs/fednnunet_requirements.txt
```

## 4. Submit FL jobs

You do **not** activate the venv yourself before `sbatch`. `scripts/slurm/fl_fednnunet.sbatch` does it on the GPU node:

1. finds the repo root from where you submitted (repo root or `scripts/slurm`),
2. `module load GPU/Python/3.13.5-bare-gcc-2025b-eb` and activates `ENV_DIR` (default `/nobackup/proj/disk/fedmed/shared/envs/fednnunet`),
3. sets `nnUNet_raw`, `nnUNet_preprocessed`, `FL_RUNS_ROOT`, `FEDNNUNET_DIR`,
4. gives each client one of the job's GPUs (extra GPUs Slurm hands out stay unused), picks a port from the job ID, splits CPUs over the data-augmentation workers,
5. prints node, GPUs, commits and package versions, then runs `scripts/run_fednnunet.sh`.

Job resources in the file: 1 node, `--gpus=3`, 36 CPUs, 128 GB, 2 h, account `naiss2026-3-793-gpu`.

Settings you pass on the command line:

| Variable | Default | Meaning |
|---|---|---|
| `CLIENTS` | `201 218 204` | dataset IDs, one client each. **Always set it** for the binary/25% runs. |
| `FOLD` | `0` | `1` = the 25% training subset |
| `SMOKE` | `1` | `1` = 2 rounds into `<RUN_NAME>_smoke`; `0` = full run |
| `RUN_NAME` | `top3_mc_fednnunet` | run folder name under `FL_RUNS_ROOT` |
| `TRAINER` | `nnUNetTrainer_250epochs` | must match the rounds (1 round = 1 epoch) |
| `ENV_DIR`, `PYTHON_MODULE` | see above | if your venv or module differ |

One-time:
```bash
mkdir -p ~/Run/slurm_logs
cd /home/skitti/code/Baseline_nnUNet_BraTS/scripts/slurm
```

Smoke test first (2 rounds, about an hour):
```bash
FOLD=1 CLIENTS="101 118 104" RUN_NAME=top3_bi_d25_fednnunet sbatch fl_fednnunet.sbatch
```
It passed if the `.out` ends with `done` and `fl_runs/top3_bi_d25_fednnunet_smoke/` has `global_final.pt`, `global_model/` and a 2-line `round_log.jsonl`.

Full runs (250 rounds):
```bash
SMOKE=0 FOLD=1 CLIENTS="101 118 104" RUN_NAME=top3_bi_d25_fednnunet sbatch --time=3-00:00:00 fl_fednnunet.sbatch
SMOKE=0 FOLD=1 CLIENTS="401 418 404" RUN_NAME=top3_highr1_d25_fednnunet sbatch --time=3-00:00:00 fl_fednnunet.sbatch
```
Use the smoke run's time per round x 250 to check the time limit.

GPU allocation: a job asking for `--gpus=3` was given all 4 GPUs of the node. Check what a job holds with `scontrol show job <jobid> | grep -i AllocTRES`. If it is `gres/gpu=4`, the 4th GPU belongs to that job and no other job can use it.

## 5. Monitor and troubleshoot

```bash
squeue -u $USER                                              # PD = waiting, R = running
squeue -u $USER --start                                      # estimated start
sacct -u $USER -S today --format=JobID,JobName,State,Elapsed,ExitCode
scancel <jobid>
```

Logs:

| Log | Where | Shows |
|---|---|---|
| Job output | `~/Run/slurm_logs/fl_fednnunet_<jobid>.out` | node, GPUs, versions, run start/end |
| Job errors | `~/Run/slurm_logs/fl_fednnunet_<jobid>.err` | module/env/dataset/fold errors |
| Server | `<FL_RUNS_ROOT>/<RUN_NAME>[_smoke]/server.log` | rounds, aggregation |
| Clients | `<FL_RUNS_ROOT>/<RUN_NAME>[_smoke]/client_<ID>.log` | each site's nnU-Net training, the real error if a client stops |

```bash
tail -f ~/Run/slurm_logs/fl_fednnunet_<jobid>.out
tail -n 50 /nobackup/proj/disk/fedmed/shared/BraTS21/fl_runs/top3_bi_d25_fednnunet_smoke/client_101.log
```

A run folder that already has `server.log` / `client_*.log` is never overwritten: rename it or use a new `RUN_NAME` before resubmitting.

Errors seen so far:

| Error | Cause | Fix |
|---|---|---|
| `The following module(s) are unknown: "GPU/Miniforge/..."` | no Miniforge module on the GPU partition | use the venv setup above (current sbatch) |
| `cannot execute binary file: Exec format error` | ARM env run on the x86 login node | run it on a GPU node (`interactive` / `srun` / `sbatch`) |
| `No module named flwr` | Flower missing from the env | `python -m pip install "flwr==1.39.0"` on a GPU node |
| `3 clients but 4 GPUs in GPUS='0 1 2 3'` | Slurm gave the whole node | fixed in the sbatch: one GPU per client from the job's GPUs |
| No `.out`/`.err` at all | `~/Run/slurm_logs` missing | `mkdir -p ~/Run/slurm_logs` |
| `Connection closed by ... port 22` at login | that login node down/busy, or IP temporarily blocked | try `arrhenius1.hpc.arrhenius.naiss.se` or `arrhenius3...`; `ssh -v` to see where it stops |
| `module load` "not unloaded: hpc_sysenv, hpc" | system modules `purge` keeps | harmless |

## 6. NAISS references

- [Arrhenius Quick Start](https://hpc.pages.naiss.se/user-documentation/support-docs/arrhenius_hpc/quickstart/): partitions, `GPU/` modules, storage quotas, Slurm
- [Login](https://hpc.pages.naiss.se/user-documentation/support-docs/arrhenius_hpc/login/login/): host names, 2FA, host keys
- [Programming environment](https://hpc.pages.naiss.se/user-documentation/support-docs/arrhenius_hpc/software_development/programming_environment/): buildenv modules (CPU and GPU) and their contents
- [Building applications](https://hpc.pages.naiss.se/user-documentation/support-docs/arrhenius_hpc/software_development/building_applications/): build GPU software in a GPU allocation, `interactive -p gpu --gpus 1`
- [Using containers](https://hpc.pages.naiss.se/user-documentation/support-docs/arrhenius_hpc/software_development/containers/): Apptainer, aarch64 rebuilds
- [Python best-practice guide: modules](https://hpc.pages.naiss.se/training/python-best-practice-guide/python/modules/): `GPU/Python` bare/bundle modules, Miniforge, "no buildenv in batch jobs"
- [PyTorch on Arrhenius](https://hpc.pages.naiss.se/user-documentation/support-docs/applications/pytorch/arrhenius/): PyTorch modules, NCCL plug-in for multi-node
- [Miniforge on Arrhenius](https://hpc.pages.naiss.se/user-documentation/support-docs/applications/miniforge/arrhenius/)
- [AI intro: Arrhenius basics](https://hpc.pages.naiss.se/training/ai-intro/lessons/basics/): GH200 hardware, conda vs modules vs containers, `virtualenv --system-site-packages`
- [AI intro: running DL](https://hpc.pages.naiss.se/training/ai-intro/lessons/running_dl/): PyTorch module load commands
- [Applications list](https://hpc.pages.naiss.se/user-documentation/support-docs/applications/)
- [Arrhenius resource page](https://www.naiss.se/resource/arrhenius/)
- [Support for Arrhenius (SUPR)](https://supr.naiss.se/support/?centre_resource=r101)

