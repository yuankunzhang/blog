---
title: "Running Alpine Linux in QEMU"
date: 2020-03-14T09:41:41+08:00
draft: false
---

How to run a Linux operating system in QEMU.

<!--more-->

## QEMU: a very short introduction

According to its [official site](https://www.qemu.org), QEMU is a generic and open source machine emulator and virtualizer.

QEMU works in one of the two operating modes:

* **Full system emulation**. In this mode, QEMU emulates a full system, including processors and peripherals. It can be used to launch virtual guest operating systems.
* **User mode emulation**. In this mode, QEMU can launch processes that were compiled for a different instruction set.

Futhermore, you can provision the `-enable-kvm` option to leverage Linux KVM. With this option, QEMU deals with the setting up and migration of KVM images and emulates hardwares, and the execution of the guest is done by KVM as requested by QEMU.

QEMU supports a long [list of disk image formats](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-using_qemu_img-supported_qemu_img_formats), including the most popular two:

* Raw images (`.img`). This can be the fastest file-based format. If your file system supports holes, then only the written sectors will reserve space. Use `qemu-img info` or `ls -ls` to obtain the real size used by the image. Although raw images give optimal performance, only very basic features are available.
* QEMU copy on write (`.qcow2`). This is the most versatile format with advanced feature set. But the feature set comes at the cost of performance.

## Run Alpine Linux in QEMU

Let's start by downloading the Alpine Linux installation media.

```sh
wget http://dl-cdn.alpinelinux.org/alpine/v3.11/releases/x86_64/alpine-standard-3.11.3-x86_64.iso
```

Create a virtual hard drive for the Linux machine. We declare the size to be 10G, but note that only the written sectors will reserve space. The actual size of this file is much smaller.

```sh
qemu-img create -f qcow2 alpine.qcow2 10G
```

That's all we need for preparation. Now let's move on to boot the installation media.

```sh
qemu-system-x86_64 \
    -enable-kvm \
    -m 2048 \
    -smp cores=2,threads=4 \
    -nic user \
    -drive file=alpine.qcow2,media=disk \
    -cdrom alpine-standard-3.11.3-x86_64.iso
```

* `-enable-kvm`: Make use of KVM when running a target architecture that is the same as the host architecture. The guest machine can then take advantage of the KVM acceleration.
* `-m 2048`: Allocate 2GB memory to guest machine.
* `-smp cores=2,threads=4`: Specify the number of CPU cores and threads to use.
* `-nic user`: Add a virtual network interface controller to guest machine. More in this [article](https://www.qemu.org/2018/05/31/nic-parameter/).
* `-drive file=alpine.qcow2,media=disk`: Attach the newly created virtual hard drive to guest machine. The virtual hard drive will be mounted at `/dev/vda`.
* `-cdrom alpine-standard-3.11.3-x86_64.iso`: Attach a virtual CDROM drive and load Alpine Linux installation media into it.

After installing Alpine Linux to the hard drive, we can boot without the `-cdrom` option.

```sh
qemu-system-x86_64 \
    -enable-kvm \
    -m 2048 \
    -nic user \
    -drive file=alpine.qcow2,media=disk
```

I'm impressed by the flexibility and versatility of QEMU (of course this may come as a downside to some people because there are tons of options to choose).
