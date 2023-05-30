+++
title = "Setting Up a Development Machine"
date = 2021-07-07T20:50:45+08:00
slug = "dev-machine-setup"
categories = ["linux"]
tags = ["linux", "arch", "bluetooth"]
draft = false
+++

Just a few days ago, I completed a [home server build](/posts/home-server-setup). Today I decided to have a clean OS reinstallation on my development machine due to several reasons:

- Firstly, the root partition on this machine is nearing full capacity (thank you, my past self).
- Secondly, I'm consistently encountering `__common_interrupt: 1.55 No irq handler for vector` errors during system bootup. There's a thread in the Arch Forum discussing this issue. Despite my efforts to resolve it through various ombinations of kernel parameters as suggested in the thread, I have not succeeded. It appears that the only viable solution is a BIOS firmware upgrade.
- Finally, there exists a paradoxical issue: I cannot decrypt the root partition without using my bluetooth keyboard, yet I cannot use my bluetooth keyboard until the root partition is decrypted and mounted (because the bluetooth driver sits in the root partition). This presents a typical chicken and egg problem. My current hack around is to use a wired USB keyboard to decrypt the root parition. However, I desperately want to abandon this workaround.

![Keyboard not found, press any key to continue...](/img/keyboard-not-found-press-any-key.png)

<!--more-->

## BIOS Firmware Upgrade

**Caution: flashing the motherboard BIOS is a high risk operation. Please ensure you have thoroughly read and understand the motherboard upgrade instructions before proceeding.**

The [Gigabyte X570 Aorus Master](https://www.gigabyte.com/Motherboard/X570-AORUS-MASTER-rev-11-12#kf) motherboard that I'm using offers a convenient [Q-Flash utility](https://www.gigabyte.com/MicroSite/121/tech_qflash.htm) embedded in the ROM, which greatly simplifies the BIOS upgrading process. All I have to do is to download the latest firmware onto a USB drive and follow the instructions in the Q-Flash manual.

![Gigabyte X570 Aorus Master is a powerful motherboard for AMD processors](/img/x570-aorus-master.png)

## Operating System Reinstallation

The installation process is not much different from [the OS installation on my home server](/posts/home-server-setup#arch-linux-installation), except that there is no need for a RAID configuration or remote disk unlocking.

However, I do want to have my bluetooth keyboard operational before the root partition is decrypted and mounted, thereby eliminating the need for a separate wired keyboard solely for the purpose of root partition decryption.

## Activating Bluetooth Service Prior to Disk Decryption

To use bluetooth devices at this early stage, We need to bring the bluetooth service into the initramfs. Luckily, a [mkinitcpio hook](https://github.com/irreleph4nt/mkinitcpio-bluetooth) designed for this specific purpose exists, thanks to the resourceful Arch Linux Community.

A few things to note about this hook:

* It does not work in conjunction with the `systemd` hook in `/etc/mkinitcpio.conf`.
* The project README says that it has only been tested on installations that use [rEFInd](https://wiki.archlinux.org/title/REFInd) as the boot loader. I'm using [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot), formerly known as gummiboot, and this hook functions perfectly fine in this environment.

We want the bluetooth adapter to power up automatically post-boot. Therefore, I need to add the line `AutoEnable=true` to the `Policy` section in the bluetooth adapter configuration file `/etc/bluetooth/main.conf`:

```conf
# In /etc/bluetooth/main.conf
[Policy]
AutoEnable=true
```

Of course, the initial pairing of the bluetooth keyboard with the system needs to be performed manually. This is a one-time requirement. From the subsequent reboot onward, my keyboard will be fully functional just before I'm prompted to enter the disk decryption key.
