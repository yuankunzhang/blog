+++
title = "Setting up a Home Server"
date = 2021-06-21T20:50:45+08:00
slug = "home-server-setup"
categories = ["linux"]
tags = ["linux", "arch", "luks", "raid"]
draft = false
+++

Recently, I came into possession of an old computer. Instead of letting it gather dust in the corner, I thought it would be a good idea to repurpose it as a home server. Primarily, I wanted a Samba Server, but I may go further to run other self-hosted services like NextCloud.

In this article I'll walk through my home server setup process.

<!--more-->

## Operating System

As a long time user and fan of Arch Linux for my desktop operating system ([and you should also try it](https://bbs.archlinux.org/viewtopic.php?id=115942)), Though it's not often to hear people using Arch Linux on a server, I'd like to take the chance and give it a try. There are several features of Arch Linux that I found particularly compelling:

- **Minimal out-of-the box system**. By default, Arch Linux comes with almost no preinstalled software, giving you the freedom to choose what to install.
- **Rolling release**. Unlike other distributions such as Debian or CentOS, Arch Linux employs a rolling release model, which means my server will always be running the most recent version of all software. However, this could introduce unstable software updates, so it's worth noting that this approach might be too aggressive for many use cases. For my home server, the risk is acceptable.
- **Outstanding Wiki**. Almost every system operation you might want to perform with your system has a detailed guide, which is a testament to the dedication of the community.

![The Linux Rolling Relase Model - https://www.maketecheasier.com/linux-rolling-release-model/](/img/linux-rolling-release-model.png.webp)

## Hardware

Here's the rundown of my hardware components.

- Motherboard: Gigabyte B360M AORUS Gaming 3
- CPU: Intel Core i5-8400 (6 cores, 12 threads)
- RAM: Vengeance LPX 8GB DDR4 DRAM 2400MHz x 4 (32GB in total)
- SSD: Samsung 980 PRO 1TB PCIe NVMe Gen4 SSD M.2
- HDD: Seagate IronWolf 4TB x 2
- GPU: None, unnecessary for this server.

## Goals

I've set some specific goals for this server build.

- **No proprietary software**. The only exception is the boot firmware. I'll give [Libreboot](https://libreboot.org/) a try later some time.
- **Full disk encryption**. The only exception is the boot partition.
- **Disk decryption via a remove SSH session**. I want to avoid the hassle of having to plug in a keyboard and a screen every time I need to restart the server.
- **RAID 1 on the two HDDs**. It will be used as the Samba storage partition.

I'm still deliberating on whether to set up the SSD as a cache for the RAID. The gain for my use case will be negligible. For now, it's not in my immediate plans, but it may be a fascinating subject for future investigation.

## Disk Partitioning and Encryption

![](https://gitlab.com/cryptsetup/cryptsetup/wikis/luks-logo.png)

Disk partitioning and encryption need to be handled before installing the operating system.

The 1TB SSD are split into two partitions: a 1GB EFI system partition (mounted at `/boot`), and the root partition taking up the rest of the SSD.

```bash
Device           Start        End    Sectors   Size Type
/dev/nvme0n1p1    2048    2099199    2097152     1G EFI System
/dev/nvme0n1p2 2099200 1953525134 1951425935 930.5G Linux filesystem
```

The root partition is encrypted by [dm-crypt](https://wiki.archlinux.org/title/Dm-crypt).

```bash
# Encrypt the root partition.
$ cryptsetup luksFormat /dev/nvme0n1p2

# Open the encrypted root partition and mount it to /dev/mapper/root.
$ cryptsetup open /dev/nvme0n1p2 root
```

The two HDDs are set up as software RAID (level 1), managed using [mdadm](https://wiki.archlinux.org/title/RAID#Installation).

```bash
$ mdadm --create --verbose --level=1 --metadata=1.2 --raid-devices=2 /dev/md0 /dev/sda1 /dev/sdb1
```

Here, `/dev/md0` is the logical RAID block device, which is also encrypted by dm-crypt.

```bash
# Encrypt the RAID block device.
$ cryptsetup luksFormat /dev/md0

# Open the RAID block device and mount it to /dev/mapper/nas.
$ cryptsetup open /dev/md0 nas
```

Below is the final layout of my disk drivers:

```bash
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   3.6T  0 disk
└─sda1        8:1    0   3.6T  0 part
  └─md0       9:0    0   3.6T  0 raid1
    └─nas   254:1    0   3.6T  0 crypt /nas
sdb           8:16   0   3.6T  0 disk
└─sdb1        8:17   0   3.6T  0 part
  └─md0       9:0    0   3.6T  0 raid1
    └─nas   254:1    0   3.6T  0 crypt /nas
nvme0n1     259:0    0 931.5G  0 disk
├─nvme0n1p1 259:1    0     1G  0 part  /boot
└─nvme0n1p2 259:2    0 930.5G  0 part
  └─root    254:0    0 930.5G  0 crypt /
```

mdadm requires the `mdadm_udev` hook, so we need to add it to the `HOOKS` section in `/etc/mkinitcpio.conf`:

```bash
$ cat /etc/mkinitcpio.conf | grep '^HOOKS'
HOOKS=(base udev autodetect modconf block mdadm_udev filesystems keyboard fsck systemd)
```

## Arch Linux Installation

Follow [this great Arch wiki](https://wiki.archlinux.org/title/installation_guide) to install the system.

I'm using [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) as the UEFI boot manager. Ensure the `CFG Lock` BIOS switch is disabled. The installation process is quite straightforward:

1. Mount the EFI system partition at `/boot`.
2. Execute `bootctl install` to install systemd-boot. This will copy systemd-boot to the EFI partition and set systemd-boot as the default EFI boot entry loaded by the EFI Boot manager.
3. Generate the bootloader entry. Two files are required: `/boot/loader/loader.conf` and `/boot/loader/entries/arch.conf`.

```bash
$ cat <<EOF > /boot/loader/loader.conf
default arch.conf
timeout 3
console-mode max
editor no
EOF

$ cat <<EOF > /boot/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=/dev/mapper/root
EOF
```

## Root Partition Decryption and Mounting

One of my objectives is to remote unlock the LUKS-encrypted root partition. For this, I use a mkinitcpio hook called [systemd-tool](https://github.com/random-archer/mkinitcpio-systemd-tool), which provides early remote SSH access before the root partition gets mounted.

First, install systemd-tool and its dependencies.

```bash
$ pacman -S mkinitcpio-systemd-tool busybox cryptsetup openssh tinyssh tinyssh-convert mc
```

Add the `systemd-tool` hook in `/etc/mkinitcpio.conf`. The final `HOOKS` section in `/etc/mkinitcpio.conf` looks like below.

```txt
$ cat /etc/mkinitcpio.conf | grep '^HOOKS'
HOOKS=(base udev autodetect modconf block mdadm_udev filesystems keyboard fsck systemd systemd-tool)
```

Following this, I need to configure `/etc/crypttab` and `/etc/mkinitcpio-systemd-tool/config/crypttab`:

```bash
$ echo "crypt UUID=$(blkid -s UUID -o value /dev/sdX2) none luks" > /etc/crypttab
$ cat /etc/crypttab > /etc/mkinitcpio-systemd-tool/config/crypttab
```

As well as `/etc/fstab` and `/etc/mkinitcpio-systemd-tool/config/fstab`:

```bash
$ echo "UUID=$(blkid -s UUID -o value /dev/mapper/root) /   ext4    rw,relatime 0 1" > /etc/fstab
$ echo "/dev/mapper/root    /sysroot    auto    x-systemd.device-timeout=9999h  0 1" > /etc/mkinitcpio-systemd-tool/config/fstab
```

Then, enable the required services and build initramfs.

```bash
$ systemctl enable initrd-cryptsetup.path
$ systemctl enable initrd-tinysshd
$ systemctl enable initrd-debug-progs
$ systemctl enable initrd-sysroot-mount

$ mkinitcpio -P
```

## Remote Unlocking

After rebooting, the server now has a fancy remote shell environment running before the root partition gets mounted. This allows me to connect to the server remotely. How are a few things to note about this remote shell environment:

- It's a tinyssh service.
- Tinyssh only recognizes Ed25519 SSH key. Therefore, I have to generate an Ed25519 key pair on my local machine and paste the public key to `/root/.ssh/authorized_keys`.
- At this early stage, we can only connect as `root` user, because other users are not yet available.

After entering the decryption key at the prompt, the root partition gets decrypted and mounted automatically.

I have not yet discussed the decryption and mounting of the RAID device. It is actually simpler. We can create a key file under `/etc/cryptsetup-keys.d/` (let's call it `nas.key`) and use this key file to decrypt the RAID device. All I need to do is modify `/etc/crypttab` and `/etc/fstab` as described below and we are good to go.

```bash
$ echo "nas UUID=$(blkid -s UUID -o value /dev/md0) /etc/cryptsetup-keys.d/nas.key" >> /etc/crypttab
$ echo "UUID=$(blkid -s UUID -o value /dev/mapper/nas) /nas   ext4    rw,relatime 0 1" >> /etc/fstab
```
