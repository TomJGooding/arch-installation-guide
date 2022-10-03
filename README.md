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
reflector --country "United Kingdom" --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
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
reflector --country "United Kingdom" --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
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

## Add users and groups

Add a new user
```sh
useradd -m -G sys,network,lp,wheel tom
```

Set the password
```sh
passwd tom
```

Enable the wheel group for sudo, by uncommenting the `%wheel ALL= (ALL) ALL` line in sudoers
```sh
EDITOR=vim visudo 
```

## Enable services

```sh
systemctl enable avahi-daemon
```

```sh
systemctl enable bluetooth
```

```sh
systemctl enable cups
```

```sh
systemctl enable fstrim.timer
```

```sh
systemctl enable NetworkManager
```

## Reboot

Exit the chroot environment
```sh
exit
```

Unmount all the partitions
```sh
umount -R /mnt
```

Restart the machine. Remember to remove the USB!
```sh
reboot
```

## Post-Installation

Ensure a valid IP address
```sh
ip -c a
```

Check you can synchronise packages
```sh
sudo pacman -Sy
```

Download the list of post-install packages file
```sh
curl -O https://raw.githubusercontent.com/TomJGooding/arch-installation-guide/main/paclists/PostInstall.paclist.txt
```

Install the packages
```sh
sudo pacman -S --needed - < PostInstall.paclist.txt
```

Enable the display manager
```sh
sudo systemctl enable lightdm
```

Edit lightdm config to use slick-greeter
```sh
sudo vim /etc/lightdm/lightdm.conf
```
```
greeter-session=lightdm-slick-greeter
```

## Install AUR packages

Install the paru AUR helper
```sh
git clone https://aur.archlinux.org/paru
cd paru
makepkg -si
```

Verify paru is working
```sh
paru
```

Remove build directory
```sh
cd
rm -fr paru
```

Download the list of AUR packages file
```sh
curl -O https://raw.githubusercontent.com/TomJGooding/arch-installation-guide/main/paclists/AUR.paclist.txt
```

Install the packages
```sh
paru -S - < AUR.paclist.txt
```

## Activate Zram

Enable zramd

```sh
sudo systemctl enable --now zramd
```

Check system now has swap with zram
```sh
lsblk
```

## Reboot and Initial Configuration

Reboot the system
```sh
reboot
```

If everything went well, after rebooting we should be greeted with the lightdm display manager.

After logging in, we will be prompted by i3 to create a configuration file. Hit enter.

Select the windows key as the modifier and hit enter.

Hit mod+Enter to open the terminal. Change the keyboard layout
```sh
setxkbmap gb
```

The urxvt terminal is ugly out of the box, so we'll need to add an Xresources file
```sh
curl -O https://raw.githubusercontent.com/TomJGooding/arch-installation-guide/main/settings/.Xresources
```

Load the file
```sh
xrdb ~/.Xresources
```

Hit mod+Shift+q to close the current terminal, then open again with mod+Enter.
We should now have a much nicer looking terminal!

## Configure Snapper

Become root shell and change to the root directory
```sh
sudo -s
cd /
```

Snapper will re-create the snapshots subvolume when configured, so we need to remove this first
```sh
umount /.snapshots
rm -r /.snapshots
```

Create the snapper root configuration
```sh
snapper -c root create-config /
```

Listing the btrfs subvolumes will show that snapper has created an extra superflous snapshots subvolume
```sh
btrfs subvol lis /
```

Remove the extra superflous subvolume
```sh
btrfs subvol del /.snapshots
```

Listing again should show this has now been deleted
```sh
btrfs subvol lis /
```

Now we need to re-create this directory and re-mount
```sh
mkdir .snapshots
mount -a
```

Check that all subvolumes are properly mounted again
```sh
lsblk
```

We want bootable snapshots, but currently the default is the the top level file system tree
```sh
btrfs subvol get-def /
```

Change the default to the @ root subvolume ID
```sh
btrfs subvol get-def /
btrfs subvol set-def ID_of_@_subvolume /
```

Verify the changes
```sh
btrfs subvol get-def /
```

Edit the config file
```sh
vim /etc/snapper/configs/root
```
```
ALLOW_GROUPS="wheel"
[...]
NUMBER_LIMIT="10"
[...]
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="0"
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_YEARLY="0"
```

Change the owner of the snapshots subvolume for the wheel group
```sh
chown -R :wheel /.snapshots
```

Exit the root shell and ensure we can use snapper as a wheel user
```sh
exit
snapper ls
```

Become root shell again and change to the root directory
```sh
sudo -s
cd /
```

Enable the grub-btrfs service
```sh
systemctl enable --now grub-btrfs.path
```

Verify that the service is monitoring for new snapshots
```sh
systemctl status grub-btrfs.path
```

Create a grub configuration file to ensure everything is synchronised
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

Enable the snapper services
```sh
systemctl enable --now snapper-timeline.timer
systemctl enable --now snapper-cleanup.timer
```

Create our first snapshot for the base system configuration
```sh
snapper -c root create -d "***Base System Configuration***"
snapper ls
```

Synchronise the grub menu
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

Exit the root shell and verify that as a wheel user the new snapshot is listed
```sh
exit
snapper ls
```
