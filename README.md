# ArchLinux-Workflow
This repository will contain my actual ArchLinux workflow for everyday use. Starting with my installation process.

## Installation guide

The first step is to check the actual file system and find the drive name.

```
lsblk
```

If it's a nvme drive, you can use the **nvme format** command to clear it.

Here the command will perform a cryptographic erasure on all the namespaces :

```
nvme format /dev/disk -s 2 -n 0xffffffff
```



