---
title: "Debug Linux Kernel With QEMU and GDB"
date: 2020-03-20T09:31:33+08:00
draft: true
toc: false
images:
tags: 
  - linux
  - qemu
  - gdb
---

In last post we see how to run a raw Linux kernel in QEMU. QEMU offers another fancy feature: it can start a GDB Server and external GDB Debugger to connect. With this we can build a comfortable environment to debug system kernels and firmware. Let's see how to leverage this feature to debug the Linux kernel.

<!--more-->

## Compiling the Kernel with Debug Info

First thing we need to do is to prepare a kernel with debug info. Enter the TUI kernel configuration interface:

```sh
$ cd linux-source/
$ make menuconfig
```

Enter "Kernel hacking > Compile-time checks and compiler options", and enable these two options:

- Compile the kernel with debug info
- Provide GDB scripts for kernel debugging

![Provide GDB scripts for kernel debugging](/img/provide-gdb-scripts-for-kernel-debugging.png)

Save the new configuration and compile the kernel by invoking `make -j8`. After the compilation, we are interested in two newly created files:

- vmlinux. This is the Linux Kernel in an statically linked executable file format with all debugging information.
- scripts/gdb/vmlinux-gdb.py. This is the GDB script for kernel debugging.

Let's add the GDB script to the GDB init file so that the script gets loaded everytime we start GDB Debugger:

```sh
$ echo "add-auto-load-safe-path `pwd`/scripts/gdb/vmlinux-gdb.py" >> ~/.gdbinit
```

## Start a Debugging Session

QEMU provides two important options for debugging purpose:

- The `-S` option prevents the CPU from starting. This gives time for debugger to connect and allows to start debugging from the very beginning.
- The `-s` option starts a GDB Server on port 1234. Later in GDB Debugger we can connect to it with `target remote :1234`.

Now let's boot the kernel with these options:

```sh
$ qemu-system-x86_64 \
    -S \
    -s \
    -enable-kvm \
    -kernel bzImage \
    -smp cores=1,threads=2 \
    -m 1024 \
    -append "console=ttyS0 nokaslr selinux=0 debug" \
    -initrd initramfs.img \
    -serial stdio \
    -display none
```

Note that the running of the process is feezed, because we have told QEMU to wait for debugger by using the `-S` option.

In another terminal, start the GDB Debugger and connect it to QEMU:

```sh
$ gdb vmlinux
Type "apropos word" to search for commands related to "word"...
Reading symbols from vmlinux...
(gdb) target remote :1234
Remote debugging using :1234
0x000000000000fff0 in exception_stacks ()
```

Now we are able to set break points and trace the running of the kernel as if it is just a normal user application:

```sh
(gdb) b start_kernel
Note: breakpoints 1 and 2 also set at pc 0xffffffff829e0aa8.
Breakpoint 3 at 0xffffffff829e0aa8: file init/main.c, line 786.
(gdb) c
Continuing.

Thread 1 hit Breakpoint 1, start_kernel () at init/main.c:786
786     {
...
```

## References

- [Debugging with QEMU](https://en.wikibooks.org/wiki/QEMU/Debugging_with_QEMU)
