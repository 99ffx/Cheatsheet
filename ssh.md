# SSH: Connecting to the Remote Machine

[← Back to index](README.md)

## Log in

```bash
ssh <liu-id>@<computer_name>.ad.liu.se
```

## Copy files

rsync -aH --partial --info=progress2 /local/data1/sirki47/data/BraTS21/nnUNet_preprocessed/Dataset201_LocalMC_site01 sirki47@suzuki.ad.liu.se:/local/data1/sirki47/data/BraTS21/nnUNet_preprocessed/


```bash
# Laptop → remote
scp local_file.txt <user>@<host>:~/PhD2026/Run/

# Remote → laptop
scp <user>@<host>:~/PhD2026/Run/logs/run_site04_fold0.out .

# Whole folders, only copies what changed (better for big folders)
rsync -avh --progress <user>@<host>:~/PhD2026/Run/logs/ ./logs/
```
