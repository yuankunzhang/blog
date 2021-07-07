+++
title = "Reinstall OS on My Dev Machine"
slug = "reinstall-os-on-dev-machine"
[taxonomies]
categories = ["linux"]
tags = ["linux", "arch", "bluetooth"]
+++

A few days ago, I had a [home server setup](/home-server-setup). Today I want to have a clean OS reinstallation on my development machine. The primary reason is that the root partition is almost full (due to a short-sightness when I first set the machine up). Apart from that, there are several other issues that I want to tackle:

- I got a lot of `__common_interrupt: 1.55 No irq handler for vector` errors during the system bootup. There's a thread in the Arch Forum on this issue. I tried booting up the system with different kernel parameters, but none helped. Seems like the only solution is to upgrade the BIOS firmware.
- I cannot use my bluetooth keyboard to input the root partition decryption key, because the bluetooth service is locked in the root partition. This is quite disturbing, everytime I restarted the machine, I had to plug in my USB wired keyboard just to type in the decryption key.

<!-- more -->

## Upgrade BIOS Firmware

**A word of warning: flashing motherboard BIOS is a dangerous operation. Make sure you read the motherboard manual thoroughly.**

I'm using a [Gigabyte X570 Aorus Master](https://www.gigabyte.com/Motherboard/X570-AORUS-MASTER-rev-11-12#kf) motherboard for this machine. It has a handy [Q-Flash utility](https://www.gigabyte.com/MicroSite/121/tech_qflash.htm) embedded in the ROM. The firmware upgrading process is very easy, just follow the manual.

## Reinstall the Operating System

The installation is not much different from [the OS installation on my home server](/home-server-setup#arch-linux-installation), except that I don't need a RAID and I don't need to remote unlock the disk.

But I do want to use my bluetooth keyboard to unlock the disk.

## Enable Bluetooth Keyboard before Disk Decryption

We need to bring the bluetooth service into the initramfs. Thankfully, someone has created a [mkinitcpio hook](https://github.com/irreleph4nt/mkinitcpio-bluetooth) for this purpose (kudos to the Arch Community).

Some notes on this hook:

* It does not work together with the `systemd` hook.
* The project README says that it has only been tested on installation that uses [rEFInd](https://wiki.archlinux.org/title/REFInd) as boot loader. I'm using [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) (previously called gummiboot), and this hook works just fine.

The bluetooth adapter needs to be auto powered on after boot. Add the line `AutoEnable=true` to the `Policy` section in the configuration file `/etc/bluetooth/main.conf`:

```conf
# In /etc/bluetooth/main.conf
[Policy]
AutoEnable=true
```

And I need to manually connect the bluetooth keyboard for a first time. Upon next reboot, my keyboard becomes functional just before I'm asked to input the disk decryption key.
