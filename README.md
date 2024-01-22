# ArchLinux-Workflow
This repository will contain my actual ArchLinux workflow for everyday use. Starting with my installation process.

## Installation guide

The first step is to check the actual file system and find the drive name. 
```
lsblk
```
I'm going to call it **/dev/nvme**, just replace it by your's.


If it's a nvme drive, you can use the **nvme format** command to clear it.

Here the command will perform a cryptographic erasure on all the namespaces :

```
nvme format /dev/disk -s 2 -n 0xffffffff
```

Then use fdisk to create a 512MB partition on the drive (i'll call it *esp*)
and create a second partition (*system*) with all the space you wan for your system.



