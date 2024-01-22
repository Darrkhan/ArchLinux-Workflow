# ArchLinux-Workflow
This repository will contain my actual ArchLinux workflow for everyday use. Starting with my installation process.
I'm using a btrfs filesystem nested inside an encrypted LVM volume.
You can follow the whole process or skip the encryption part if you prefer.

## Installation guide

### Disk preparation
The first step is to check the actual file system and find the drive name. 
```
lsblk
```
I'm going to call it **/dev/nvme0**, just replace it by your's.


If it's a nvme drive, you can use the **nvme format** command to clear it.

> [!CAUTION]
> In the next step all data will be erased on the selected disk, be sure to backup important files !

Here the command will perform a cryptographic erasure on all the namespaces :

```
nvme format /dev/nvme0 -s 2 -n 0xffffffff
```

Then use fdisk to create a 512MB partition on the drive (i'll call it **/dev/nvme0esp**)
and create a second partition (**/dev/nvme0system**) with as much space as you want for your system.

### Encryption
Follow this part only if you want to encrypt your main system.

We start by launching a benchmark to find most secure and fastest duo cipher/key size

```
cryptsetup benchmark
```

We then use **luksFormat** using the wanted cipher and key size. In my case it's *aes-xts* with a key size of *256* bits.

```
cryptsetup luksFormat /dev/nvme0system --cipher aes-xts-plain64 --hash sha256 --key-size 256
```

Once the disk encrypted we can now open it using the new passphrase defined just before.

```
cryptsetup luksOpen /dev/nvme0system cryptSys
```

### Filesystem creation

We start by creating the LVM physical volume:

```
pvcreate /dev/mapper/cryptSys
```

then the logical volume:

```
vgcreate cryptLVM /dev/mapper/cryptSys
```

We can now create 2 partition, one for the swap (*check how much you need before*) and one for the system.

```
lvcreate -L 16G cryptLVM -n swap
lvcreate -l 100%FREE cryptLVM -n ArchSys
```

The next step is to create the btrfs filesystem:

```
mkfs.btrfs /dev/mapper/cryptLVM-ArchSys
```

Mount it to create subvolumes

```
mount /dev/mapper/cryptLVM-ArchSys /mnt
```

With btrfs i like to use 4 subvolumes: 

> [!TIP]
> I like using this layout because i can avoid to add useless data in futur snapshots like the **/var/log** directory
    
| subvolume name  | mount point |                     command                  |
| --------------- | ----------- | -------------------------------------------- |
| @               | /           | ```btrfs subvolume create /mnt/@```          |
| @home           | /home       | ```btrfs subvolume create /mnt/@home```      |
| @snapshots      | /snapshots  | ```btrfs subvolume create /mnt/@snapshots``` |
| @varlog         | /var/log    | ```btrfs subvolume create /mnt/@varlog```    |

now that the btrfs filesystem is created, unmount it and format the efi partition:

```
umount /mnt
mkfs.fat -F32 /dev/nvme0esp
```

