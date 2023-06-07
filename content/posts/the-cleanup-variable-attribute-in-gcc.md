---
title: "The Cleanup Variable Attribute in GCC"
date: 2023-06-06T19:59:32+08:00
draft: false
tags: ["c", "gcc"]
---

C++ has a powerful idiom called [RAII](https://www.stroustrup.com/bs_faq2.html#finally) (Resource Acquisition Is Initialization)[^1], although the name might leave something to be desired. The fundamental idea is to represent a resource by a local object, tying the resource's lifecycle to the lifecycle of the local object' lifecycle. In other words, we could say the local object is the _owner_ of the resource (Hmm, I smell something rusty here). The local object is responsible for releasing the resource in its destructor. Once the local object goes out of its scope, the resource is released. This mechanism remains valid even in the event of an exception within the scope. This is one of my most wanted feature when I'm programming in C. Without it, some error handling situations often lead to numerous `goto failure` statements, where the `failure` block is merely used for releasing resources in a carefully crafted sequence. Appearently, the C standard committee is discussing to add [a defer mechanism](https://gustedt.wordpress.com/2020/12/14/a-defer-mechanism-for-c/) to C, taking inspiration from the `defer` construct in Golang.

Today I learned that GCC has already provided the semantic through an extension called the `cleanup` variable attribute. The cleanup attribute allows us to define an auto variable with a cleanup handler that will be invoked when the auto variable goes out of score. The cleanup handler must take one parameter, which is a pointer to a type compatible with the variable. The return value of the function (if any) is ignored. Below is its syntax.

```c
void handler(void *p) {
    /* cleanup logic */
}

/* Must be applied only on auto variables */
attribute __((cleanup(handler))) var;
```

## Usage

Here's tiny program that illustrates how to use the cleanup attribute. In the code, I've added a switch, `USE_CLEANUP`, which can be toggled to activate or deactivate the application of the `cleanup` variable attribute. In the `foo` function, I've initiated two pointers, `x` and `y`, to point to two int objects on the heap. Note that I didn't deallocate the two int objects in the `foo` function.

```c
#include <stdio.h>
#include <stdlib.h>

#ifdef USE_CLEANUP
#define CLEANUP __attribute__ ((cleanup(cleanup_func)))
#else
#define CLEANUP
#endif

static void cleanup_func(void *p) {
  printf("[cleanup_func] value=%d\n", **(int **)p);
  free(*(void **)p);
}

void foo() {
  CLEANUP int *x = malloc(sizeof(int));
  CLEANUP int *y = malloc(sizeof(int));
  *x = 42;
  *y = 39;
  printf("[foo] x=%d\n", *x);
  printf("[foo] y=%d\n", *y);
}

int main(void) {
  foo();
  return 0;
}
```

Now, compile the program with the `-DUSE_CLEANUP` flag and run it. We see that the cleanup handler is invoked twice, in the reverse order of how we defined the two integer pointers.

```sh
[foo] x=42
[foo] y=39
[cleanup_func] value=39
[cleanup_func] value=42
```

## Implementation

In order to implement the `cleanup` attribute, GCC inserts instructions to call the handler before returning from the current function. We can confirm this by comparing the two variants of the above program - one that compiled with the `-DUSE_CLEANUP` flag and another that does not. I've turned off stack protector to avoid nay unrelated stack protection instructions.

```sh
$ cc -DUSE_CLEANUP -fno-stack-protector -S -o with_cleanup.s main.c
$ cc -fno-stack-protector -S -o no_cleanup.s main.c
$ diff -y no_cleanup.s with_cleanup.s
```

![](/img/cleanup-diff.png)

The diff result clearly shows that in the `-DUSE_CLEANUP` version, GCC generates instructions to call the cleanup handler twice at the termination of the `foo` function. That's the only distinction between the two versions.

From this implementation, we can draw a few insights:

- The cleanup handler won't be called if the local scope executes any of the `exit` functions or the `abort` function.
- The cleanup handler might not be called or might not complete in the event of a terminate signal, depending on the timing of the signal reception.
- The cleanup handler won't be called if the local scope performs a long jump instead of a normal return.

Moreover, as per the [GCC documentation](https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html), it is undefined what happens if the cleanup handler does not return normally (e.g., if it performs a long jump).
