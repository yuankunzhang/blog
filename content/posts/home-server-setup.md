+++
title = "Setup a Home Server"
date = 2021-06-21T20:50:45+08:00
slug = "home-server-setup"
categories = ["linux"]
tags = ["linux", "arch", "luks", "raid"]
draft = false
+++

Recently I got a retired computer. Better than letting it sit in the corner queitly and become a dust collector, I've been planning to turn it into a home server. I mainly need a Samba Server, but I may go further to run other self-hosted services like NextCloud.

In this article I'll talk about my setup of this home server.

<!--more-->

## Operating System

I've been using Arch Linux as my desktop OS for a long time, and I'm loving it ([and you should also try it](https://bbs.archlinux.org/viewtopic.php?id=115942)). Though it's rare to hear people using Arch Linux as server OS, I'd like to give it a try. I will maybe write a review after six months or one year of running it. There are some shining features that I love the most about Arch Linux:

- **Minimal system out of the box**. Arch Linux by default has almost nothing installed. You choose what to install.
- **Rolling release**. This is what other distros like Debian or CentOS are missing. Because of the rolling release model, my server will be always running the latest everything. Well, it may be a concern that this rolling release model is a bit too aggressive and may introduce unstable features. As I'm using it only for my home server, the risk is acceptable.
- **Excellent Wiki**. There are detailed guides on almost everything you want to do with your system. Kudos to the community!

![The Linux Rolling Relase Model - https://www.maketecheasier.com/linux-rolling-release-model/](/img/linux-rolling-release-model.png.webp)

## Hardware

Here are my hardware components.

- Motherboard: Gigabyte B360M AORUS Gaming 3
- CPU: Intel Core i5-8400 (6 cores, 12 threads)
- RAM: Vengeance LPX 8GB DDR4 DRAM 2400MHz x 4 (32GB in total)
- SSD: Samsung 980 PRO 1TB PCIe NVMe Gen4 SSD M.2
- HDD: Seagate IronWolf 4TB x 2
- GPU: Don't have one, don't need one for this server.

## Goals

These are the goals I want to achieve with this server build.

- **No proprietary software**. Except the boot firmware for now. I'll give [Libreboot](https://libreboot.org/) a try later some time.
- **Full disk encryption**. Except the boot partition.
- **Disk decryption via a remove SSH session**. I don't want to plug in a keyboard and a screen every time I need to restart the server.
- **RAID 1 on the two HDDs**. It will be used as the Samba storage partition.

I'm not sure whether or not it's worth to setup the SSD as a cache for the RAID. For now I'm excluding it from my goals. It will be an interesting investigation for future time.

## Disk Partitioning and Encryption

![](https://gitlab.com/cryptsetup/cryptsetup/wikis/luks-logo.png)

Disk partitioning and encryption need to be done before the installation of the operating system.

The 1TB SSD are split into two partitions: 1GB of the EFI system partition (mounted at `/boot`); and the root partition taking up the rest of the SSD.

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

The two HDDs are implemented as software RAID (level 1). I'm using [mdadm](https://wiki.archlinux.org/title/RAID#Installation) to manage it.

```bash
$ mdadm --create --verbose --level=1 --metadata=1.2 --raid-devices=2 /dev/md0 /dev/sda1 /dev/sdb1
```

Here, `/dev/md0` is the logical RAID block device. It is, in turn, encrypted by dm-crypt.

```bash
# Encrypt the RAID block device.
$ cryptsetup luksFormat /dev/md0

# Open the RAID block device and mount it to /dev/mapper/nas.
$ cryptsetup open /dev/md0 nas
```

Below is the final layout of my disks:

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

Mdadm needs the `mdadm_udev` hook, so we should add it to the `HOOKS` section in `/etc/mkinitcpio.conf`:

## Arch Linux Installation

Follow [this great Arch wiki](https://wiki.archlinux.org/title/installation_guide) to get the system installed.

I'm using [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) as the UEFI boot manager. The `CFG Lock` BIOS switch must be disabled. The installation is actually very straightforward:

1. Mount the EFI system partition at `/boot`.
2. Run `bootctl install` to install systemd-boot. This will copy systemd-boot to the EFI partition, and then set systemd-boot as the default EFI boot entry loaded by the EFI Boot manager.
3. Create the bootloader entry. Two files are needed: `/boot/loader/loader.conf` and `/boot/loader/entries/arch.conf`.

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

One of the goals is for me to be able to remote unlock the LUKS-encrypted root partition. There's this mkinitcpio hook called [systemd-tool](https://github.com/random-archer/mkinitcpio-systemd-tool). It provides early remote SSH access before the root partition gets mounted.

First, install systemd-tool and its dependencies.

```bash
$ pacman -S mkinitcpio-systemd-tool busybox cryptsetup openssh tinyssh tinyssh-convert mc
```

It requires the `systemd-tool` hook. Add this hook in `/etc/mkinitcpio.conf`. The final `HOOKS` section in `/etc/mkinitcpio.conf` looks like below.

```txt
HOOKS=(base udev autodetect modconf block mdadm_udev filesystems keyboard fsck systemd systemd-tool)
```

After that, I need to configure `/etc/crypttab` and `/etc/mkinitcpio-systemd-tool/config/crypttab`.

```bash
$ echo "crypt UUID=$(blkid -s UUID -o value /dev/sdX2) none luks" > /etc/crypttab
$ cat /etc/crypttab > /etc/mkinitcpio-systemd-tool/config/crypttab
```

As well as `/etc/fstab` and `/etc/mkinitcpio-systemd-tool/config/fstab`.

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

After rebooting, the server now has a fancy remote shell environment running before the root partition gets mounted. This allows me to connect to the server remotely. A few things to note about this remote shell though:

- The shell is a tinyssh service.
- Tinyssh only recognizes Ed25519 SSH key. So I need to generate an Ed25519 key pair on my local machine and paste the public key to `/root/.ssh/authorized_keys`.
- In this early stage, we can only connect as `root` user, because other users are not available yet.

The root partition will automatically get decrypted and mounted after the decryption key is typed into the prompt.

I haven't talked about the decryption and mounting of the RAID device. It is actually less a problem. We can simply create a key file under `/etc/cryptsetup-keys.d/` (let's call it `nas.key`) and use this key file to decrypt the RAID device. Modify `/etc/crypttab` and `/etc/fstab` as described below and we are all set.

```bash
$ echo "nas UUID=$(blkid -s UUID -o value /dev/md0) /etc/cryptsetup-keys.d/nas.key" >> /etc/crypttab
$ echo "UUID=$(blkid -s UUID -o value /dev/mapper/nas) /nas   ext4    rw,relatime 0 1" >> /etc/fstab
```
