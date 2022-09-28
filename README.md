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

Final layout
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

Btrfs subvolumes layout
|     Name     |       Subvolume       |
|:------------:|:---------------------:|
| "@.snapshots |      /.snapshots      |
|     @home    |         /home         |
|     @log     |        /var/log       |
|     @pkgs    | /var/cache/pacman/pkg |
