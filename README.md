# Arch Installation Guide

Notes for installing Arch Linux

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

Identify the target disk for installation
```sh
lsblk
```

