# Stopping Jobs

[← Back to index](README.md)

## Stop by PID

```bash
kill <PID>                     # polite stop (SIGTERM)
kill -9 <PID>                  # force stop, only if the polite stop doesn't work
kill $(cat logs/run_site04_fold0.pid)   # using the saved PID file
```

## Stop by matching the command

```bash
pkill -f "train_local.sh 104 0"
```

Check what would be matched before killing:

```bash
pgrep -af "train_local.sh 104 0"
```

## Stop the script and all its children

The bash script and its python children are separate processes. Killing only the bash PID can leave python running on the GPU.

```bash
pkill -P <PID>      # kill the children of PID
kill <PID>          # then the parent
```

Then confirm the GPU is free:

```bash
nvidia-smi
```

> **Careful:** a bare `pkill -f nnUNet` kills **every** nnU-Net run of mine on that machine.
