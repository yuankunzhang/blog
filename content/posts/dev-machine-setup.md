+++
title = "My Dev Machine Setup"
date = 2021-07-07T20:50:45+08:00
slug = "dev-machine-setup"
categories = ["linux"]
tags = ["linux", "arch", "bluetooth"]
draft = false
+++

A few days ago, I had a [home server build](/posts/home-server-setup). Today I want to have a clean OS reinstallation on my development machine. The primary reason is that the root partition is almost full (due to short-sightness when I first set up this machine). Apart from that, there are several other issues that I want to tackle:

- I got a lot of `__common_interrupt: 1.55 No irq handler for vector` errors during the system bootup. There's a thread in the Arch Forum discussing this issue. I tried booting up the system with different kernel parameters, but none helped. Seems like the only solution is to upgrade the BIOS firmware.
- I cannot use my bluetooth keyboard to input decryption key to decrypt the root partition, because the bluetooth service is locked in the root partition and it's a chicken-egg problem. This is quite disturbing, everytime I restarted the machine, I had to wire in my USB keyboard just to input the decryption key.

<!--more-->

## Upgrade BIOS Firmware

**A word of warning: flashing motherboard BIOS is a dangerous operation. Make sure you read the motherboard manual thoroughly.**

I'm using a [Gigabyte X570 Aorus Master](https://www.gigabyte.com/Motherboard/X570-AORUS-MASTER-rev-11-12#kf) motherboard for this machine. It has a handy [Q-Flash utility](https://www.gigabyte.com/MicroSite/121/tech_qflash.htm) embedded in the ROM. The firmware upgrading process is very easy, just follow the manual.

## Reinstall the Operating System

The installation is not much different from [the OS installation on my home server](/posts/home-server-setup#arch-linux-installation), except that I don't need a RAID and I don't need to remote unlock the disk.

But I do want to have my bluetooth keyboard available before the root partition gets decrypted and mounted, so that I can use it to type in the decryption key.

## Enable Bluetooth Keyboard before Disk Decryption

We need to bring the bluetooth service into the initramfs. Thankfully, someone has created a [mkinitcpio hook](https://github.com/irreleph4nt/mkinitcpio-bluetooth) for this purpose (kudos to the Arch Linux Community).

Some notes on this hook:

* It does not work together with the `systemd` hook.
* The project README says that it has only been tested on installation that uses [rEFInd](https://wiki.archlinux.org/title/REFInd) as boot loader. I'm using [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) (previously called gummiboot), and this hook works just fine.

The bluetooth adapter needs to be auto powered on after boot. So I need to add the line `AutoEnable=true` to the `Policy` section in the configuration file `/etc/bluetooth/main.conf`:

```conf
# In /etc/bluetooth/main.conf
[Policy]
AutoEnable=true
```

And I need to boot the system and manually pair my bluetooth keyboard for a first time. This is a one-off task. Upon next reboot, my keyboard becomes functional just before I'm asked to input the disk decryption key.
