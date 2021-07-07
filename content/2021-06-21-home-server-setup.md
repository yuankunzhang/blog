+++
title = "My Home Server Setup"
slug = "home-server-setup"
[taxonomies]
categories = ["linux"]
tags = ["linux", "arch", "luks", "raid"]
+++

Recently I got some retired computer hardware. Better than putting it in the corner and let it absorb dust, I've been planning to turn it into a home server. I want to use it primarily as a Samba Server. It also comes handy to run some self-hosted services like NextCloud.

In this article I'll talk about the setup of this home server.

<!-- more -->

## Operating System

I've been using Arch Linux as my desktop OS for a long time, and I'm loving it. Though it's rare to hear people using Arch Linux as server OS, I'd like to give it a try. I will write a review after maybe six months or one year of running it. There are some features that I love the most about Arch Linux:

- **Minimal system out of the box**. Arch Linux by default has almost nothing installed. You choose what to install.
- **Rolling release**. This is what other distros like Debian or CentOS are missing. Because of the rolling release model, my server will be always on the latest everything. Well, it may be a concern that this rolling release model is a bit too aggressive and may introduce breaking changes. As I'm using it only for my home server, the risk is acceptable.
- **Excellent Wiki**. There are detailed guides on almost everything you want to do with your system. Kudos to the community!

## Hardware

Here are my hardware specifications.

- Motherboard: Gigabyte B360M AORUS Gaming 3
- CPU: Intel Core i5-8400 (6 cores, 12 threads)
- RAM: Vengeance LPX 8GB DDR4 DRAM 2400MHz x 4 (32GB in total)
- SSD: Samsung 980 PRO 1TB PCIe NVMe Gen4 SSD M.2
- HDD: Seagate IronWolf 4TB x 2
- GPU: Don't have one, don't need one.

## Goals

These are the goals I want to achieve with this server build.

- **No proprietary software (except the boot firmware)**. I'll give [Libreboot](https://libreboot.org/) a try later some time.
- **Full disk encryption (except the boot partition)**.
- **Disk decryption via a remove SSH session**. I don't want to plug in a keyboard and a screen every time I need to restart the server.
- **RAID 1 on the two HDDs**. It will be used as the Samba storage partition.

I'm not sure whether or not it's worth to setup the SSD as a cache for the RAID. For now I'm excluding it from my goals. It will be an interesting investigation for my future time.

## Disk Partitioning and Encryption

Disk partitioning and encryption need to be done before the installation of the operating system.

The 1TB SSD are split into two partitions: 1GB of the EFI system partition (mounted at `/boot`); and the root partition taking up the rest of the SSD.

```shell
Device           Start        End    Sectors   Size Type
/dev/nvme0n1p1    2048    2099199    2097152     1G EFI System
/dev/nvme0n1p2 2099200 1953525134 1951425935 930.5G Linux filesystem
```

The root partition is encrypted by [dm-crypt](https://wiki.archlinux.org/title/Dm-crypt).

```shell
# Encrypt the root partition.
$ cryptsetup luksFormat /dev/nvme0n1p2

# Open the encrypted root partition and mount it to /dev/mapper/root.
$ cryptsetup open /dev/nvme0n1p2 root
```

The two HDDs are implemented as software RAID (level 1). I'm using [mdadm](https://wiki.archlinux.org/title/RAID#Installation) for this purpose.

```shell
$ mdadm --create --verbose --level=1 --metadata=1.2 --raid-devices=2 /dev/md0 /dev/sda1 /dev/sdb1
```

Here, `/dev/md0` is the logical RAID block device. It is, in turn, encrypted by dm-crypt. Below is the final layout of the HDDs.

```shell
# Encrypt the RAID block device.
$ cryptsetup luksFormat /dev/md0

# Open the RAID block device and mount it to /dev/mapper/nas.
$ cryptsetup open /dev/md0 nas
```

Below is the final layout of my disks:

```shell
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

mdadm needs the `mdadm_udev` hook, we should add it to the `HOOKS` section in `/etc/mkinitcpio.conf`:

## Arch Linux Installation

Follow [the great Arch wiki](https://wiki.archlinux.org/title/installation_guide) and we are all good.

I'm using [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) as the UEFI boot manager. The `CFG Lock` BIOS switch must be disabled. The installation is actually very simple:

1. Mount the EFI system partition at `/boot`.
2. Run `bootctl install` to install systemd-boot. This will copy systemd-boot to the EFI partition, and then set systemd-boot as the default EFI boot entry loaded by the EFI Boot manager.
3. Create the bootloader entry. Two files are needed: `/boot/loader/loader.conf` and `/boot/loader/entries/arch.conf`.

```shell
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

One of the goals is for me to be able to remote unlock the LUKS-encrypted root partition. I'm using this mkinitcpio hook named [systemd-tool](https://github.com/random-archer/mkinitcpio-systemd-tool). It provides early remote SSH access before the root partition gets mounted.

First, install systemd-tool and its dependencies.

```shell
$ pacman -S mkinitcpio-systemd-tool busybox cryptsetup openssh tinyssh tinyssh-convert mc
```

It requires the `systemd-tool` hook. The final `HOOKS` section in `/etc/mkinitcpio.conf` looks like below.

```txt
HOOKS=(base udev autodetect modconf block mdadm_udev filesystems keyboard fsck systemd systemd-tool)
```

After that, I need to configure `/etc/crypttab` and `/etc/mkinitcpio-systemd-tool/config/crypttab`.

```shell
$ echo "crypt UUID=$(blkid -s UUID -o value /dev/sdX2) none luks" > /etc/crypttab
$ cat /etc/crypttab > /etc/mkinitcpio-systemd-tool/config/crypttab
```

As well as `/etc/fstab` and `/etc/mkinitcpio-systemd-tool/config/fstab`.

```shell
$ echo "UUID=$(blkid -s UUID -o value /dev/mapper/root) /   ext4    rw,relatime 0 1" > /etc/fstab
$ echo "/dev/mapper/root    /sysroot    auto    x-systemd.device-timeout=9999h  0 1" > /etc/mkinitcpio-systemd-tool/config/fstab
```

Then, enable required services and build initramfs.

```shell
$ systemctl enable initrd-cryptsetup.path
$ systemctl enable initrd-tinysshd
$ systemctl enable initrd-debug-progs
$ systemctl enable initrd-sysroot-mount

$ mkinitcpio -P
```

## Remote Unlocking

After rebooting, the server now has a fancy remote shell running before the root partition gets mounted. This allows me to connect to the server remotely. A few things to note about this remote shell:

- The shell is a tinyssh service.
- Tinyssh only recognizes Ed25519 SSH key. So I need to generate an Ed25519 key pair on my local machine and paste the public key to `/root/.ssh/authorized_keys`.
- In this early stage, we can only connect as `root` user, because other users are not available yet.

The root partition will automatically get decrypted and mounted after the decryption key is inputed into the prompt.

I haven't talked about the decryption and mounting of the RAID device. It is actually less a problem. We can simply create a key file under `/etc/cryptsetup-keys.d/` (let's call it `nas.key`) and use this key file to decrypt the RAID device. Modify `/etc/crypttab` and `/etc/fstab` as described below and we are all set.

```shell
$ echo "nas UUID=$(blkid -s UUID -o value /dev/md0) /etc/cryptsetup-keys.d/nas.key" >> /etc/crypttab
$ echo "UUID=$(blkid -s UUID -o value /dev/mapper/nas) /nas   ext4    rw,relatime 0 1" >> /etc/fstab
```
