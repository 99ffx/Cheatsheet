# Viewing an nnU-Net dataset in AFNI (image + mask overlay)

Browse a whole nnU-Net split (e.g. `imagesTs` + `labelsTs`) in one AFNI window:
all cases are stacked into one 4D file, and the **Index** control steps through
the cases with the mask following each image.

Works because BraTS images all share the same grid (240×240×155).
If cases have different grids, `3dTcat` fails → use [Alternative](#alternative-load-everything-without-stacking).

---

## 1. Stack images and masks (one case per frame)

```bash
D=/local/data1/sirki47/data/BraTS21/nnUNet_raw/Dataset101_LocalBi_site01   # dataset
SPLIT=Ts                                  # Ts or Tr
MOD=0000                                  # modality: 0000, 0001, ... (see dataset.json)
OUT=/local/data1/sirki47/afni_view        # keep OUTSIDE the nnU-Net folders
mkdir -p $OUT && cd $OUT

# Case lists in the same order (only cases that have a mask)
ls $D/labels$SPLIT/*.nii.gz | sort > cases_seg.txt
sed "s|/labels$SPLIT/\(.*\)\.nii\.gz\$|/images$SPLIT/\1_$MOD.nii.gz|" cases_seg.txt > cases_img.txt

# Stack
3dTcat -prefix ${SPLIT}_img.nii.gz $(cat cases_img.txt)
3dTcat -prefix ${SPLIT}_seg.nii.gz $(cat cases_seg.txt)
```

Disk use: ~35 MB per case for images (masks much less). Delete `$OUT` when done.

## 2. Open in AFNI

```bash
cd $OUT
afni -DAFNI_SLAVE_FUNCTIME=YES -DAFNI_SLAVE_THRTIME=YES \
     -com "SWITCH_UNDERLAY ${SPLIT}_img.nii.gz" \
     -com "SWITCH_OVERLAY ${SPLIT}_seg.nii.gz" \
     -com "SET_PBAR_ALL A.+99 4 ROI_i32" \
     -com "SET_THRESHNEW A 0" \
     ${SPLIT}_img.nii.gz ${SPLIT}_seg.nii.gz
```

| Option | What it does |
|---|---|
| `-DAFNI_SLAVE_FUNCTIME=YES` | overlay (**OLay**) frame follows the Index |
| `-DAFNI_SLAVE_THRTIME=YES` | threshold (**Thr**) frame follows the Index — without it you keep seeing the first case's mask |
| `SET_PBAR_ALL A.+99 4 ROI_i32` | distinct color per label (range 0–4 fits BraTS labels) |
| `SET_THRESHNEW A 0` | threshold 0 so all labels show |

AFNI rule: **options (`-D`, `-com`) go before the files.**

## 3. Navigate

| Action | How |
|---|---|
| Next / previous **case** | **Index** ▲▼ in the main AFNI window (top left, under the coordinates) |
| Jump to case N | type N in the Index field + Enter |
| Next / previous case (keyboard) | `]` / `[` in an image window |
| Change **slice** | Page Up / Page Down (or the slice arrows) |
| Check frames | **Define Overlay** → ULay / OLay / Thr should all show the same `#` |

If **Thr** doesn't follow (older AFNI): set Thr to the same number as OLay by hand in **Define Overlay**.

Which case is Index *i*? (Index 0 = line 1)

```bash
i=25; sed -n "$((i+1))p" $OUT/cases_seg.txt
```

## Make the settings permanent (optional)

```bash
echo "AFNI_SLAVE_FUNCTIME = YES" >> ~/.afnirc
echo "AFNI_SLAVE_THRTIME = YES"  >> ~/.afnirc
```

If `~/.afnirc` already has an `***ENVIRONMENT` section, put the lines inside it.

---

## Alternative: load everything without stacking

No disk use, but you pair image and mask by hand:

```bash
cd $D
afni -com "SET_PBAR_ALL A.+99 4 ROI_i32" -com "SET_THRESHNEW A 0" \
     -all_dsets imagesTs/*.nii.gz labelsTs/*.nii.gz
```

Pick a case with **Underlay**, then the mask with the same ID with **Overlay**.
`-all_dsets` puts files from different folders into one AFNI session.

## Troubleshooting

| Problem | Fix |
|---|---|
| Mask stays the same when changing Index | add `-DAFNI_SLAVE_THRTIME=YES` (Thr frame stuck at `#0`) |
| AFNI shows its default image | options must come **before** the files |
| `Couldn't open ... as session OR as dataset` | file path wrong / file missing — check with `ls -lL <file>` and `3dinfo <file>` |
| `3dTcat` error about grids | cases differ in size → use the Alternative |
| No window over SSH | connect with `ssh -Y` (Mac: install XQuartz) |