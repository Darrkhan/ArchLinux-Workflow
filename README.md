# ArchLinux-Workflow
This repository will contain my actual ArchLinux workflow for everyday use, starting with the installation process. 
I'm using a btrfs filesystem nested inside an encrypted LVM volume.
You can follow the entire process or skip the encryption part if you prefer.

## Installation guide

### Disk preparation
The first step is to check the actual file system and find the drive name. 
```
lsblk
```
I'm going to refer to it as **/dev/nvme0**, just replace it by your drive name.


If it's a nvme drive, you can use the **nvme format** command to clear it.

> [!CAUTION]
> The following command will perform a cryptographic erasure on all namespaces. Backup important files before proceeding!

```
nvme format /dev/nvme0 -s 2 -n 0xffffffff
```

Next, use fdisk to create a 512MB partition on the drive (let's call it **/dev/nvme0esp**) and create a second partition (**/dev/nvme0system**) with the desired system space.

### Encryption
Follow this section only if you want to encrypt your main system.

Begin by launching a benchmark to find the most secure and fastest duo cipher/key size.

```
cryptsetup benchmark
```

Use luksFormat with the chosen cipher and key size. In my case, it's *aes-xts* with a key size of *256* bits.

```
cryptsetup luksFormat /dev/nvme0system --cipher aes-xts-plain64 --hash sha256 --key-size 256
```

Once the disk is encrypted, open it using the newly defined passphrase.

```
cryptsetup luksOpen /dev/nvme0system cryptSys
```

### Filesystem creation

Start by creating the LVM physical volume:

```
pvcreate /dev/mapper/cryptSys
```

Then create the logical volume:

```
vgcreate cryptLVM /dev/mapper/cryptSys
```

Now create two partitions, one for swap (adjust size as needed) and one for the system:

```
lvcreate -L 16G cryptLVM -n swap
lvcreate -l 100%FREE cryptLVM -n ArchSys
```

Create the btrfs filesystem:

```
mkfs.btrfs /dev/mapper/cryptLVM-ArchSys
```

Mount it to create subvolumes:

```
mount /dev/mapper/cryptLVM-ArchSys /mnt
```

With btrfs, I prefer using four subvolumes:

> [!TIP]
>This layout helps avoid adding unnecessary data in future snapshots, such as the **/var/log** directory.
    
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

### Mount and install the system

Start by mounting the root partition and then creating all the needed mountpoints

> [!NOTE]
> I use generic mounting option for btrfs subvolumes, if you don't understand this well just use these ones.


```
mount -o noatime,discard,ssd,compress=zstd,subvol=@ /dev/mapper/cryptLVM-ArchSys /mnt
mkdir -p /mnt/home /mnt/boot /mnt/var/log /mnt/.snapshots
```

then mount all the subvolumes:

```
mount -o noatime,discard,ssd,compress=zstd,subvol=@home /dev/mapper/cryptLVM-ArchSys /mnt/home
mount -o noatime,discard,ssd,compress=zstd,subvol=@snapshots /dev/mapper/cryptLVM-ArchSys /mnt/.snapshots
mount -o noatime,discard,ssd,compress=zstd,subvol=@varlog /dev/mapper/cryptLVM-ArchSys /mnt/var/log
```

mount and initialize the swap partition:

```
mkswap /dev/mapper/cryptLVM-swap
swapon /dev/mapper/cryptLVM-swap
```

then mount the efi partition on **/mnt/boot**:

```
mount /dev/nvme0esp /mnt/boot
```

Update mirror list with reflector:

> [!NOTE]
> Use mirrors from your country and the nearest one to have a faster download speed.

```
reflector --country France --country Germany --latest 10 --sort rate --protocol https --save /etc/pacman.d/mirrorlist
```

Use the pacstrap command to install the main system:

```
pacstrap -K /mnt base base-devel linux linux-firmware git nano ntp networkmanager btrfs-progs man-db lvm2 man-pages texinfo wget openssh dosfstools bluez-utils blueman pipewire-jack pipewire-alsa pipewire-pulse pipewire-audio wireplumber
```

Generate the fstab file:

```
genfstab -U /mnt >> /mnt/etc/fstab
```

now chroot inside the system:

```
arch-chroot /mnt
```

Setup the system, use your own parameters when needed:

```
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
nano /etc/locale.gen (uncomment wanted locales)
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=fr" > /etc/vconsole.conf
```

Set hostname and localhost:

> [!NOTE]
> Replace ArchSus by the hostname you want.

```
echo "ArchSus" > /etc/hostname

echo -e "127.0.0.1   localhost\n::1         localhost\n127.0.1.1   ArchSus.localdomain ArchSus" > /etc/hosts
```

Enable network manager to start it on reboot :

```
systemctl enable NetworkManager
```

Then setup the bootloader: 

> [!IMPORTANT]
> I'm using grub because of the possibility to boot on a btrfs snapshot thanks to grub-btrfs.

```
pacman -S amd-ucode
pacman -S grub grub-btrfs efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=MainOS
```

Add **encrypt**, **lvm2** and **btrfs** hooks in mkinitcpio:

```
sed -i 's/HOOKS=(base udev autodetect modconf kms keyboard keymap block filesystems fsck)/HOOKS=(base udev autodetect modconf kms keyboard keymap block encrypt lvm2 btrfs filesystems fsck)/' mkinitcpio.conf
```

Use blkid to find and replace **<DISK_UUID>** by your real UUID:

```
sed -i 's/loglevel=3 quiet/cryptdevice=UUID=<DISK_UUID>:cryptSys:allow-discards root=\/dev\/cryptLVM\/ArchSys rw rootflags=subvol=@ loglevel=3 quiet/' /etc/default/grub

grub-mkconfig -o /boot/grub/grub.cfg
```

You're now ready to reboot: 

```
exit
umount -R /mnt
reboot
```

### System not booting correctly after installation

> [!IMPORTANT]
> Keep in mind that instead of reinstalling everything, if you didn't mess up the partition, you can repair it by chrooting back inside the system.

To chroot back inside the system:

```
cryptsetup luksOpen /dev/nvme0system cryptSys
vgscan
vgchange -ay cryptLVM
mount -o subvol=@ /dev/mapper/cryptLVM-ArchSys /mnt
mount /dev/nvme0esp /mnt/boot
arch-chroot /mnt
```

> [!TIP]
> Double-check the UUID or LVM volume name in the GRUB_CMDLINE_LINUX_DEFAULT.


