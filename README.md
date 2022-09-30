# Arch Installation Guide

Personal reference guide for installing Arch Linux. Disclaimer: YMMV!

## Features

- Btrfs
- zram
- i3
- Snapper

## Pre-Installation

Download an Arch Linux image and create a bootable USB. 

Connect an Ethernet cable.

Boot from the USB into the live environment.

Set the console keyboard layout (default is US).
```sh
loadkeys uk
```

Verify the boot mode
```sh
ls /sys/firmware/efi/efivars
```

Ensure a valid IP address
```sh
ip -c a
```

Verify internet connection
```sh
ping archlinux.org
```

Update the system clock
```sh
timedatectl set-ntp true
```

## Partition the Disk

Identify the target disk
```sh
lsblk
```

Use gdisk to modify partition tables
```sh
gdisk /dev/the_disk_to_be_partitioned
```

Partition layout reference
| No |    Size   | Type Code | Name |
|:--:|:---------:|:---------:|:----:|
|  1 |   +512M   |    ef00   | BOOT |
|  2 | remainder |  default  | ROOT |

For each partition
- Command: `n`
- Partition number (default): `<enter>`
- First sector (default): `<enter>`
- Last sector: *size*
- Code: *type_code*
- Command: `c`
- Partition number: *no*
- Enter name: *name*

Print the partition table to ensure configured correctly with `p`

Write the changes to the disk with `w`

Show the partitions
```sh
lsblk
```

## Format the Partitions

Format the BOOT (EFI) partition
```sh
mkfs.vfat -n BOOT /dev/boot_partition
```

Format the ROOT partition with the btrfs filesystem
```sh
mkfs.btrfs -L ROOT /dev/root_partition
```

## Mount the Partitions

Show the partitions
```sh
lsblk
```

Mount the root volume
```sh
mount /dev/root_partition /mnt
```

Change into mnt directory
```sh
cd /mnt
```

Btrfs subvolumes layout reference
|     Name     |       Subvolume       |
|:------------:|:---------------------:|
| @.snapshots  |      /.snapshots      |
|     @home    |         /home         |
|     @log     |        /var/log       |
|     @pkgs    | /var/cache/pacman/pkg |

Create the btrfs subvolumes
```sh
btrfs su cr @
btrfs su cr @home
btrfs su cr @.snapshots
btrfs su cr @log
btrfs su cr @pkgs
```

Change back into main directory
```sh
cd
```

Unmount root and then remount

```sh
umount /mnt
mount -o compress=zstd:1,noatime,subvol=@ /dev/root_partition /mnt
```

Create the directories for the subvolumes
```sh
mkdir -p /mnt/{boot/efi,home,.snapshots,var/log,var/cache/pacman/pkg}
```

Mount the btrfs subvolumes
```sh
mount -o compress=zstd:1,noatime,subvol=@home /dev/root_partition /mnt/home
mount -o compress=zstd:1,noatime,subvol=@.snaphots /dev/root_partition /mnt/.snaphots
mount -o compress=zstd:1,noatime,subvol=@log /dev/root_partition /mnt/var/log
mount -o compress=zstd:1,noatime,subvol=@pkgs /dev/root_partition /mnt/var/cache/pacman/pkg
```

Mount the EFI boot partition
```sh
mount /dev/boot_partition /mnt/boot/efi
```

Check the partitons are mounted correctly
```sh
lsblk
```

## Installation

Select the mirrors
```sh
reflector --country United Kingdom --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
```

Enable parallel downloads by uncommenting the ParallelDownloads line in the configuration file
```sh
vim /etc/pacman.conf
```

Sync the pacman repository
```sh
pacman -Syy
```

Install the essential packages listed in the `Essential.paclist.txt` file
```sh
pacstrap /mnt base linux linux-firmware other_essential_packages
```

## Configure the system

Generate an fstab file
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

Change root into the new system
```sh
arch-chroot /mnt
```

Set the time zone
```sh
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```

Synchronise the hardware clock to the system clock
```sh
hwclock --systohc
```

Edit /etc/locale.gen and uncomment the needed locales - en_GB.UTF-8 UTF-8
```sh
vim /etc/locale.gen
```
Generate the locales
```sh
locale-gen
```

Create the locale.conf file and set the LANG variable
```sh
echo "LANG=en_GB.UTF-8" >> /etc/locale.conf
```

Set the console keymap permanently
```sh
echo "KEYMAP=uk" >> /etc/vconsole.conf
```

Create the hostname
```sh
echo "arch" >> /etc/hostname
```

Edit the hosts file
```sh
vim /etc/hosts
```
```
127.0.0.1        localhost
::1              localhost
127.0.1.1        arch.localdomain        arch
```

Set the root password
```sh
passwd
```

Edit the configuration file for mkinitcpio
```sh
vim /etc/mkinitcpio.conf
```
```
BINARIES=(btrfs)
```

Regenerate
```sh
mkinitcpio -p linux
```

## Install base packages

Select the mirrors
```sh
reflector --country United Kingdom --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
```

Enable parallel downloads by uncommenting the ParallelDownloads line in the configuration file
```sh
vim /etc/pacman.conf
```

Sync the pacman repository
```sh
pacman -Syy
```

Download the list of base packages file
```sh
curl -O https://raw.githubusercontent.com/TomJGooding/arch-installation-guide/main/paclists/Base.paclist.txt
```

Install the packages
```sh
pacman -S --needed - < Base.paclist.txt
```

## Install GRUB

Install GRUB
```sh
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux
```

Create the GRUB configuration file
```sh
grub-mkconfig -o /boot/grub/grub.cfg 
```


