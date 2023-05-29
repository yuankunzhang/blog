---
title: "A Guide on Running Alpine Linux in QEMU"
date: 2020-03-14T09:41:41+08:00
draft: false
tags:
  - linux
  - qemu
---

How to run a Linux operating system in QEMU.

<!--more-->

## A Very Brief Introduction to QEMU

[QEMU](https://www.qemu.org) (Quick EMUlator) is a generic and open source machine emulator originally developed by [Fabrice Bellard](https://bellard.org/) (this same guy is also the original author of FFMPEG, insanely). It is a powerful and versatile tool that has gained widespread recognition and adoption in the world of virtualization. It provides a platform for creating and managing virtual machines, enabling users to emulate various hardware architectures, operating systems, and software environments.

One of the notable features of QEMU is its ability to provide full system emulation, allowing users to run unmodified guest operating systems on different host platforms. Additionally, QEMU supports a variety of virtualization techniques, such as hardware-assisted virtualization (e.g., Intel VT-x, AMD-V) and software based emulation, providing a range of options for running virtual machines efficiently.

QEMU also includes a suite of user-level emulation modes, enabling users to run software applications compiled for a different architecture without the need for complete system emulation. This feature is particularly useful for software developers who need to test their applications on different platforms without the overhead of running a full virtual machine.

Furthermose, QEMU integrates with the popular KVM hypervisor on Linux, allowing it to take advantages of hardware virtualization extensions and deliver near-native performance for virtualized workloads. This combination of QEMU and KVM has become a robust and widely adopted solution for virtualization in cloud computing environments.

## Running Alpine Linux in QEMU

Let's start by downloading the Alpine Linux installation media.

```sh
ALPINE_VERSION=v3.11.3
wget "http://dl-cdn.alpinelinux.org/alpine/${ALPINE_VERSION%.*}/releases/x86_64/alpine-standard-${ALPINE_VERSION}-x86_64.iso"
```

Next, we craft a virtual hard drive for out Linux virtual machine. QEMU supports a long [list of image formats](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-using_qemu_img-supported_qemu_img_formats). Among those, qcow2 is the most versatile format with the best feature set. We are going to use qcow2 format for our virtual machine.

We declare the size to be 10G, but note that only the written sectors will reserve space. The actual size of this file is much smaller.

```sh
$ qemu-img create -f qcow2 alpine.qcow2 10G
```

Having ticked off the preparation checklist, we can now boot the installation media.

```sh
$ qemu-system-x86_64 \
    -enable-kvm \
    -m 2048 \
    -smp cores=2,threads=4 \
    -nic user \
    -drive file=alpine.qcow2,media=disk \
    -cdrom alpine-standard-3.11.3-x86_64.iso
```

The options are explained as:

* `-enable-kvm`: Utilizes KVM when running a target architecture that mirrors the host architecture, allowing the guest machine to take advantage of KVM acceleration.
* `-m 2048`: Allocates 2GB memory to the guest machine.
* `-smp cores=2,threads=4`: Specify the number of CPU cores and threads to utilize.
* `-nic user`: Appends a virtual network interface controller to the guest machine. Learn more in this [article](https://www.qemu.org/2018/05/31/nic-parameter/).
* `-drive file=alpine.qcow2,media=disk`: Attaches the newly created virtual hard drive to guest machine. The virtual hard drive will be mounted at `/dev/vda`.
* `-cdrom alpine-standard-3.11.3-x86_64.iso`: Attaches a virtual CDROM drive and loads the Alpine Linux installation media into it.

Post the Alpine installation onto the hard drive, booting can be proceed without the `-cdrom` option.

```sh
$ qemu-system-x86_64 \
    -enable-kvm \
    -m 2048 \
    -nic user \
    -drive file=alpine.qcow2,media=disk
```
