---
title: "Linux Syscall Table Generator"
date: 2023-06-04T20:50:14+08:00
draft: false
tags: ["linux", "syscall"]
---

This weekend, I worked on a script that scans through the Linux source tree and generates syscall tables for both x86 and x64 architectures. If you are interested, you can check it out in [this repo](https://github.com/yuankunzhang/syscall-table-generator). I'd like to share some notes about it.

The generated tables are here:

- [Linux x64 syscall table](https://yuankun.me/syscall64/)
- [Linux x86 syscall table](https://yuankun.me/syscall32/)

In Linux, syscalls are identified by numbers, and their parameters are in machine word size, either 32-bit or 64-bit. Each system call can have up to 6 parameters, and both the syscall number and the parameters are stored in specific registers. On the x64 architecture, the syscall number is stored in the `%rax` register, while parameters are stored in `%rdi`, `%rsi`, `%rdx`, `%r10`, `%r8`, `%r9` registers, in that order. On the x86 architecture, the sycall number is stored in the `%eax` register, while parameters are stored in `%ebx`, `%ecx`, `%edx`, `%esi`, `%edi`, `%ebp` registers, in that order.

In the Linux source tree, the syscalls are defined using one of the `SYSCALL_DEFINE` macros. Let's take the `getpid` syscall defined in `kernel/sys.c` as an example:

```c
SYSCALL_DEFINE0(getpid)
{
	return task_tgid_vnr(current);
}
```

The number at the end of the macro name indicates the number of parameters for the syscall. In this case, the `getpid` syscall has zero parameters. The range for this number is from 0 to 6, as each syscall can have up to 6 parameters.

Now, let's consider another example, the `read` syscall defined in `fs/read_write.c`:

```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	return ksys_read(fd, buf, count);
}
```

The macro name indicates that this syscall has 3 parameters. Inside the parenthesis, the first symbol, `read`, is the syscall name, the following symbols indicate the type and name of each parameter, alternating in that pattern.

This gives up the basic idea about how to search for all syscall definitions in the source tree. Let's try using the following `grep` command:

```sh
$ grep "\bSYSCALL_DEFINE[0-6]\b" **/*.c
arch/alpha/kernel/osf_sys.c:SYSCALL_DEFINE1(osf_brk, unsigned long, brk)
arch/alpha/kernel/osf_sys.c:SYSCALL_DEFINE4(osf_set_program_attributes, unsigned long, text_start,
arch/alpha/kernel/osf_sys.c:SYSCALL_DEFINE4(osf_getdirentries, unsigned int, fd,
...
```

Unfortunately, many syscall definitions span multiple lines, such as the second and the third definitions in the above output. We want to capture the entire definition starting `SYSCALL` to the closing parenthesis. I ended up with the following `grep` pattern:

```sh
$ grep -Poz "(?s)\bSYSCALL_DEFINE[0-6]\b\(.*?\)" **/*.c
```

Some explanation on the `grep` flags used above:

- The `-z` flag instructs `grep` to process the content of each input file as a whole, rather than line by line. `grep` adds a trailing NUL character, instead of a newline.
- The `-o` flag tells grep to only print the matching part of the input. Since the `-z` flag is used, each input file is now treated like a single long line.
- The `-P` flag enables `grep` to use Perl-compatible regular expressions (PCREs). We need this because we utilize the `(?s)` marker, which activates the `PCRE_DOTALL` mode. This mode allows the dot symbol to match any character, including newlines.

We find all syscall definitions and store them in a temporary file named `/tmp/all_defs`:

```sh
$ grep -Poz "(?s)\bSYSCALL_DEFINE[0-6]\b\(.*?\)" **/*.c |
    tr -d '\n'        |  # merge lines
    tr '\0' '\n'      |  # fix the universe by converting NUL characters (introduced by the `-z` flag) to newlines.
    > /tmp/all_defs
```

Note that most syscalls are architecture-independent. However, a few syscalls are defined in architecture-specific code files, as they entail specific machine characteristics. For instance, the `rt_sigreturn` syscall is defined multiple times across different architectures:

```sh
$ grep '\bSYSCALL_DEFINE[0-6](rt_sigreturn' /tmp/all_defs
arch/arc/kernel/signal.c:SYSCALL_DEFINE0(rt_sigreturn)
arch/arm64/kernel/signal.c:SYSCALL_DEFINE0(rt_sigreturn)
arch/csky/kernel/signal.c:SYSCALL_DEFINE0(rt_sigreturn)
arch/powerpc/kernel/signal_32.c:SYSCALL_DEFINE0(rt_sigreturn)
arch/powerpc/kernel/signal_64.c:SYSCALL_DEFINE0(rt_sigreturn)
arch/riscv/kernel/signal.c:SYSCALL_DEFINE0(rt_sigreturn)
arch/s390/kernel/signal.c:SYSCALL_DEFINE0(rt_sigreturn)
arch/x86/kernel/signal_64.c:SYSCALL_DEFINE0(rt_sigreturn)
```

For the scope of this syscall table generator, we only concentrate on the x86 and x64 architectures. We'll eliminate definitions of other architectures by feeding the preceding result to another `grep` command:

```sh
$ grep '\bSYSCALL_DEFINE[0-6]' /tmp/all_defs |
    grep -v "arch/[^x]"      |  # eliminate non-x86 architectures	
```

This certainly improves the result, but there are a couple of remaining issues:

- We need to discard definitions under `arch/x86/um/*.c`, since those definitions are exclusive for User-Mode Linux.
- The source files `arch/x86/kernel/process_32.c` and `arch/x86/kernel/process_64.c` are mutual exclusive. We should retain only one of these. We'll keep the 32-bit.

Here's the modified command:

```sh
$ grep '\bSYSCALL_DEFINE[0-6]' /tmp/all_defs |
    grep -v "arch/[^x]"      |  # eliminate non-x86 architectures	
    grep -v "arch/x86/um"    |  # eliminate User-Mode Linux
    grep -v "process_64.c"      # eliminate 64-bit
```

Another issue arises with the syscalls `clone` and `sigsuspend`, each having multiple definitions sitting within conditional macros.

Here's `clone` in file `kernel/fork.c`:

```c
#ifdef __ARCH_WANT_SYS_CLONE
#ifdef CONFIG_CLONE_BACKWARDS
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 unsigned long, tls,
		 int __user *, child_tidptr)
#elif defined(CONFIG_CLONE_BACKWARDS2)
SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#elif defined(CONFIG_CLONE_BACKWARDS3)
SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,
		int, stack_size,
		int __user *, parent_tidptr,
		int __user *, child_tidptr,
		unsigned long, tls)
#else
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#endif
{
	struct kernel_clone_args args = {
		.flags		= (lower_32_bits(clone_flags) & ~CSIGNAL),
		.pidfd		= parent_tidptr,
		.child_tid	= child_tidptr,
		.parent_tid	= parent_tidptr,
		.exit_signal	= (lower_32_bits(clone_flags) & CSIGNAL),
		.stack		= newsp,
		.tls		= tls,
	};

	return kernel_clone(&args);
}
#endif
```

And `sigsuspend` in file `kernel/signal.c`:

```c
#ifdef CONFIG_OLD_SIGSUSPEND
SYSCALL_DEFINE1(sigsuspend, old_sigset_t, mask)
{
	sigset_t blocked;
	siginitset(&blocked, mask);
	return sigsuspend(&blocked);
}
#endif
#ifdef CONFIG_OLD_SIGSUSPEND3
SYSCALL_DEFINE3(sigsuspend, int, unused1, int, unused2, old_sigset_t, mask)
{
	sigset_t blocked;
	siginitset(&blocked, mask);
	return sigsuspend(&blocked);
}
#endif
```

As these two syscalls are dependant on the kernel configuration at the build time, we can't predict which definitions will be compiled into the kernel binary. The best we can do is to arbitrarily select one. We'll pick the first one.

```sh
$ grep '\bSYSCALL_DEFINE[0-6]' /tmp/all_defs |
    grep -v "arch/[^x]"      |  # eliminate non-x86 architectures	
    grep -v "arch/x86/um"    |  # eliminate User-Mode Linux
    grep -v "process_64.c"   |  # eliminate 64-bit
	head -1
```

Having isolated all the syscall definitions for x64 or x86, our remaining task is to present them in a table format and apply final polishing.