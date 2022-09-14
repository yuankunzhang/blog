+++
title = "Setup a Development Machine"
date = 2021-07-07T20:50:45+08:00
slug = "dev-machine-setup"
categories = ["linux"]
tags = ["linux", "arch", "bluetooth"]
draft = false
+++

A few days ago, I did a [home server build](/posts/home-server-setup). Today I want to have a clean OS reinstallation on my development machine, because of a few things:

- The root partition on this machine is almost full (thank you, my two years ago self).
- I got a lot of `__common_interrupt: 1.55 No irq handler for vector` errors during the system bootup. There's a thread in the Arch Forum discussing this issue. I tried booting up the system with several different combinations of kernel parameters mentioned in the thread, but none helped. It seems like the only working solution is to upgrade the BIOS firmware.
- I cannot decrypt the root partition without using my bluetooth keyboard, in the meantime I cannot use my bluetooth keyboard before the root partition is decrypted and mounted (because the bluetooth service sits in the root partition). This is a chicken-egg problem. My current hack around is to use a wired USB keyboard to decrypt the root parition. But I desperately want to eliminate the use of this USB keyboard.

![Keyboard not found, press any key to continue...](/img/keyboard-not-found-press-any-key.png)

<!--more-->

## Upgrade BIOS Firmware

**A word of warning: flashing motherboard BIOS is a dangerous operation. Make sure you read the motherboard manual thoroughly.**

I'm using a [Gigabyte X570 Aorus Master](https://www.gigabyte.com/Motherboard/X570-AORUS-MASTER-rev-11-12#kf) motherboard for this machine. It has a handy [Q-Flash utility](https://www.gigabyte.com/MicroSite/121/tech_qflash.htm) embedded in the ROM that simplifies the entire BIOS upgrading task. All I have to do is to download the latest firmware into a USB driver and follow the Q-Flash manual.

![Gigabyte X570 Aorus Master is a powerful motherboard for AMD processors](/img/x570-aorus-master.png)

## Reinstall the Operating System

The installation process is not much different from [the OS installation on my home server](/posts/home-server-setup#arch-linux-installation), except that I don't need a RAID and I don't need to remote unlock the disk.

But I do want to have my bluetooth keyboard available before the root partition gets decrypted and mounted, so I don't need to use a second wired keyboard only for the purpose of root partition decryption.

## Enable Bluetooth Service before Disk Decryption

To be able to use bluetooth devices at this stage, We need to bring the bluetooth service into the initramfs. Thankfully, someone has created a [mkinitcpio hook](https://github.com/irreleph4nt/mkinitcpio-bluetooth) exactly for this purpose (kudos to the Arch Linux Community again).

Some notes on this hook:

* It does not work together with the `systemd` hook in `/etc/mkinitcpio.conf`.
* The project README says that it has only been tested on installation that uses [rEFInd](https://wiki.archlinux.org/title/REFInd) as boot loader. I'm using [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) (previously called gummiboot), and this hook works just fine.

We want the bluetooth adapter to be auto powered after boot. So I need to add the line `AutoEnable=true` to the `Policy` section in the bluetooth adapter configuration file `/etc/bluetooth/main.conf`:

```conf
# In /etc/bluetooth/main.conf
[Policy]
AutoEnable=true
```

And of course to pair the bluetooth keyboard with the system for the first time, I need to do it manually. This is a one-off task. Upon next reboot, my keyboard becomes functional just before I'm asked to input the disk decryption key.
