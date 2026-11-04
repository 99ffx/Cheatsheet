# nnU-Net (v2)

[← Back to index](README.md)

## Environment variables

nnU-Net needs these three paths. Set them in `~/.bashrc` **and** inside every job script (detached shells may not read `~/.bashrc`).

```bash
export nnUNet_raw=/path/to/nnUNet_raw
export nnUNet_preprocessed=/path/to/nnUNet_preprocessed
export nnUNet_results=/path/to/nnUNet_results
```

Check that they are set:

```bash
echo $nnUNet_raw $nnUNet_preprocessed $nnUNet_results
```

## Plan and preprocess

```bash
nnUNetv2_plan_and_preprocess -d <DATASET_ID> --verify_dataset_integrity
```

## Train

```bash
nnUNetv2_train <DATASET_ID> 3d_fullres <FOLD> -tr <TRAINER>
# example
nnUNetv2_train 104 3d_fullres 0 -tr nnUNetTrainer_250epochs
```

| Option | Meaning |
|---|---|
| `<DATASET_ID>` | Dataset number, e.g. `104` |
| `3d_fullres` | Configuration (`2d`, `3d_fullres`, `3d_lowres`, `3d_cascade_fullres`) |
| `<FOLD>` | `0`–`4`, or `all` to train on all data |
| `-tr` | Trainer class (default `nnUNetTrainer` = 1000 epochs) |
| `--c` | Continue from the latest checkpoint after a crash |
| `--npz` | Save softmax outputs (needed for ensembling / best-config search) |

## Available trainers

Variants like `nnUNetTrainer_250epochs` change the number of epochs. List the training-length variants installed:

```bash
ls "$(python -c 'import nnunetv2, os; print(os.path.dirname(nnunetv2.__file__))')/training/nnUNetTrainer/variants/training_length/"
```

Search all trainers by name:

```bash
grep -rhoE "class nnUNetTrainer\w*" \
  "$(python -c 'import nnunetv2, os; print(os.path.dirname(nnunetv2.__file__))')/training/nnUNetTrainer/" \
  | sort -u
```

## Results folder

```
$nnUNet_results/<DatasetXXX_Name>/<TRAINER>__nnUNetPlans__3d_fullres/fold_<FOLD>/
├── training_log_*.txt       # nnU-Net's own log
├── progress.png             # loss and Dice curves
├── checkpoint_latest.pth    # saved regularly during training
├── checkpoint_best.pth      # best validation so far
├── checkpoint_final.pth     # only exists once training has finished
└── validation/summary.json  # validation metrics after training
```

Quick check whether a fold has finished:

```bash
ls $nnUNet_results/<DatasetXXX_Name>/<TRAINER>__nnUNetPlans__3d_fullres/fold_*/checkpoint_final.pth
```

## Resume after a crash

```bash
nnUNetv2_train 104 3d_fullres 0 -tr nnUNetTrainer_250epochs --c
```

### Resume training

```bash
CUDA_VISIBLE_DEVICES=0 nohup nnUNetv2_train 218 3d_fullres 0 -tr nnUNetTrainer_250epochs --c > logs/local_Dataset218_LocalMC_site18_resume.log 2>&1 &
```

## Predict

```bash
nnUNetv2_predict -i <INPUT_FOLDER> -o <OUTPUT_FOLDER> -d <DATASET_ID> \
    -c 3d_fullres -f 0 -tr <TRAINER>
```

Use `-f 0 1 2 3 4` to ensemble all five folds.
