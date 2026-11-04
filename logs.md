# Watching Logs

[← Back to index](README.md)

## Follow and read

```bash
tail -f logs/<RUN_NAME>.out          # follow live output (Ctrl+C only stops watching, not the job)
tail -n 50 logs/<RUN_NAME>.out       # last 50 lines (check here whether it finished or crashed)
less +F logs/<RUN_NAME>.out          # scrollable and live; Ctrl+C to scroll, F to follow again, q to quit
```

## Search for errors

```bash
grep -i -n "error\|traceback\|out of memory" logs/<RUN_NAME>.out
grep -l -i "traceback" logs/*.out    # which log files contain a crash
```

## Follow progress only

```bash
grep -i "epoch" logs/<RUN_NAME>.out | tail -n 5
```

## Find the newest log

```bash
ls -lt logs/ | head
tail -f "$(ls -t logs/*.out | head -n 1)"
```

## nnU-Net's own training log

nnU-Net also writes its own log, the loss curve and checkpoints in the results folder.
See [nnunet.md](nnunet.md#results-folder).

```bash
ls $nnUNet_results/<DatasetXXX_Name>/<TRAINER>__nnUNetPlans__3d_fullres/fold_<FOLD>/
tail -f $nnUNet_results/<DatasetXXX_Name>/<TRAINER>__nnUNetPlans__3d_fullres/fold_<FOLD>/training_log_*.txt
```

## Clean up old logs

```bash
mkdir -p logs/archive && mv logs/*_old*.out logs/archive/
```
