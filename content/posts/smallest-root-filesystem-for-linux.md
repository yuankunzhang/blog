---
title: "Smallest Root Filesystem for Linux"
date: 2020-05-22T19:40:33+08:00
draft: false
toc: false
images:
tags: 
  - linux
  - rootfs
  - init
---

The Linux kernel works hand in hand with what is called the root filesystem. This is the filesystem which will be mounted to the root directory and where the user space applications are located. How tiny can a root filesystem be, while still remaining functional? By functional, I mean it is capable of letting the kernel finish its booting process and pass control to user space without causing a kernel panic.

In a previous post, [Running the Raw Linux Kernel in QEMU](/posts/running-the-raw-linux-kernel-in-qemu), we discussed that many Linux distributions make use of two kinds of root filesystems: the initial RAM-based root filesystem and the real root filesystem. The former serves a purpose to decrypt, mount, and run the latter. In another post, [Debugging the Linux Kernel with QEMU and GDB](debugging-the-linux-kernel-with-qemu-and-gdb), we used only the initial root filesystem without providing a real one. What happens if we remove the initial root filesystem from the qemu command as well? The kernel will panic when attempting to mount the root filesystem.

```sh
$ qemu-system-x86_64 \
    -enable-kvm \
    -kernel bzImage \
    -append "console=ttyS0 nokaslr selinux=0 debug" \
    -serial stdio \
    -display none
...
[   10.366676] Call Trace:
[   10.366765]  <TASK>
[   10.366946]  dump_stack_lvl+0x4a/0x80
[   10.367210]  panic+0x185/0x340
[   10.367295]  mount_block_root+0x299/0x2a0
[   10.367393]  prepare_namespace+0xf0/0x170
[   10.367493]  kernel_init_freeable+0x38c/0x3f0
[   10.367596]  ? __pfx_kernel_init+0x10/0x10
[   10.367697]  kernel_init+0x15/0x1b0
[   10.367776]  ret_from_fork+0x2c/0x50
[   10.367914]  </TASK>
[   10.368224] Kernel Offset: disabled
[   10.368500] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

Consequently, we must provide a root filesystem and tell the kernel where to locate it. Once the kernel finds and mounts the root filesystem, it attempts to launch the first user space process (the "init" process) by executing a file found on this root filesystem. There are three ways for the kernel to find the file:

1. Initially, it tries to use the `init=` kernel parameter specified at boot time.
2. if the parameter is not set, the kernel then tries to use the value of the `CONFIG_DEFAULT_INIT` kernel build option, if it was present during the kernel build.
3. If all these fail, the kernel tries to find the file in a list of fallback locations. This list includes `/sbin/init`, `/etc/init`, `/bin/init`, and `/bin/sh`. If still unsuccessful, the kernel panics.

Here is the kernel code that handles the launch of the init process.

```c
/* In file init/main.c */
static int __ref kernel_init(void *unused)
{
  /* ... */

	/*
	 * We try each of these until one succeeds.
	 *
	 * The Bourne shell can be used instead of init if we are
	 * trying to recover a really broken machine.
	 */
	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		panic("Requested init %s failed (error %d).",
		      execute_command, ret);
	}

	if (CONFIG_DEFAULT_INIT[0] != '\0') {
		ret = run_init_process(CONFIG_DEFAULT_INIT);
		if (ret)
			pr_err("Default init %s failed (error %d)\n",
			       CONFIG_DEFAULT_INIT, ret);
		else
			return 0;
	}

	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;

	panic("No working init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/admin-guide/init.rst for guidance.");
}
```

The concept is straightforward: we just need to place an executable binary under the path `/sbin/init` of the root filesystem, and the kernel will find it and launch it. Here's a simple such program, it starts an infinite loop to echo whatever the user types on the console. Note that this program must be compiled using the `-static` GCC option.

```c
/*
 * filename: init.c
 * compile: cc -o init -static init.c
 */
#include <stdio.h>

int main(void) {
  size_t n = 0;
  char *line = NULL;

  while (1) {
    fputs("[minroot] $ ", stdout);
    if (getline(&line, &n, stdin) != -1)
      fputs(line, stdout);
    else
      clearerr(stdin);
  }
}
```

Now we can construct the root filesystem:

```sh
# Create an image file and format it.
truncate -s1M minroot
mkfs.ext2 minroot

# Mount the image as a loopback device,
# and copy over our init executable.
sudo mount -oloop minroot /mnt
sudo mkdir /mnt/sbin
sudo cc -o /mnt/sbin/init -static init.c
sudo umount /mnt
```

Let's repeat the above qemu command, but this time, we provide the root filesystem image to it as a hard drive.

```sh
$ qemu-system-x86_64 \
    -enable-kvm \
    -kernel bzImage \
    -hda minroot \
    -append "root=/dev/sda console=ttyS0 nokaslr selinux=0 debug" \
    -serial stdio \
    -display none
...
[    9.753474] VFS: Mounted root (ext2 filesystem) readonly on device 8:0.
[    9.755672] devtmpfs: error mounting -2
[    9.809455] Freeing unused kernel image (initmem) memory: 4616K
[    9.809906] Write protecting the kernel read-only data: 26624k
[    9.814056] Freeing unused kernel image (rodata/data gap) memory: 1664K
[    9.961361] x86/mm: Checked W+X mappings: passed, no W+X pages found.
[    9.961949] Run /sbin/init as init process
[    9.962048]   with arguments:
[    9.962118]     /sbin/init
[    9.962203]     nokaslr
[    9.962266]   with environment:
[    9.962422]     HOME=/
[    9.962492]     TERM=linux
[minroot] $
```

We've successfully reached the infinite echoing loop. Although our init process is not doing anything practically useful, it demonstrates how the kernel passes the torch to user space. In front of the init process is the vast and intriguing universe of user space.

I'll end this post with a question: What will happen if the init process exits/aborts?
