---
title: "The first kernel module that I wrote"
date: "2021-10-11"
type: "article"
---

The first program I ever wrote was a C program to print "Hello, world" on stdout; and that was exactly what my first kernel module did. It wrote "Hello world", but this time to the kernel log buffer. This module did one extra thing though, it wrote "Goodbye world" to the kernel log buffer when removed from the kernel memory.

Alright! lets go ahead and look at this kernel module...

```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>

MODULE_AUTHOR("Rajat Asthana");
MODULE_DESCRIPTION("My first ever kernel module!");
MODULE_LICENSE("GPL");

static int __init helloworld_init(void) {
	printk(KERN_INFO "Hello, world\n");
	return 0;
}

static void __exit helloworld_exit(void) {
	printk(KERN_INFO "Goodbye World\n");
}

module_init(helloworld_init);
module_exit(helloworld_exit);
```

Now lets break it down into some broad sections and understand what is being done in each of them.

## Section 1 — Kernel Headers

So, the first 3 lines, unlike our usual user space C code, these are kernel headers. you can even locate them in your linux machines. Just check the contents of the directory

 `/lib/modules/$(uname -r)`. Here is how it looks on my system:

![Untitled](https://raw.githubusercontent.com/rast-7/blog/2063e8c1dbe573d10e212a55154052762d75faa9/content/articles/The%20First%20Kernel%20Module%20that%20I%20wrote%2046ae6d53b6c047da9b865306bfc8a195/Untitled.png)

The soft link, `build`  underlined with yellow line, it points to the location where these headers reside. Now, one question might arise, how does our module accesses them? So, to answer that, we describe in our makefile where to look for the header, so when we compile the module, the compiler resolves by searching in the location where these headers reside.

## Section 2 — Module Macros

Next, we have three module macros of the form `MODULE_FOO()`. They just do what you might have guessed...

- `MODULE_AUTHOR()` — Specifies the author of the module
- `MODULE_DESCRIPTION()` — Describes the function of the module
- `MODULE_LICENSE()` — Specifies the license under which the module is released

These are information which the end user of your module might want. And how do they read that info, by going through my module's source code 🤔 ?? Nope! There is a command just for that and that `modinfo`. So, they use this command, and it displays all these information to them — no need to go through the source code.

## Section 3 — Entry and Exit points

You might be wondering that there is no `main()` function in the module. Well, our module is not an application and hence it doesn't have it's entry point, i.e `main()`. If you notice the last 2 lines of the kernel module, 

```c
module_init(helloworld_init);
module_init(helloworld_exit);
```

these are macros specifying our entry and exit points respectively. Both of them take a function pointer. Hence, in our code `helloworld_init` function is the entry point and `helloworld_exit` is the exit point. 

When I say entry and exit points, I mean in the sense of constructors and destructors as we have in object oriented programming (don't confuse, this is not actually OOP code, I was just giving an example). When our module is loaded in the kernel memory, the function that is called is the one passed as parameter to `module_init()` macro, in our case that happens to be `helloworld_init()` and similarly when our module is removed from the kernel memory, the function that is called is the one passed as a parameter to `module_exit()` macro, in our case that happens to be `helloworld_exit()` function.

Also, note the return type of the `helloworld_init()` function; it returns an `int`. So, when we are successful in whatever we want to do in the init section, we return `0` else we return `-E` (where E is an integer that represents the kind of error, you can find them in the kernel header files `include/uapi/asm-generic/errno-base.h`.

## Section 4 — The __init and __exit keywords

Well when I came across the kernel modules for the first time, the only thing that irked me were these keywords. I am sure, if you are also looking it for the first time, you might also be puzzled.

Alright! so they are macros and they are memory optimization attributes inserted by the linker.

The `__init` macro defines an `init.text` section for code. The idea is that whatever is there in the `init` section is used up when the module is loaded in the kernel memory (during initialization). Once it is invoked it won't be called again; so once it is called; it is then freed up.

`__exit` is similar to `__init`. Once the module is removed from the kernel memory cleanup happens, and the things defined in the function having the `__exit` macro is used up and the memory is freed up!

**Question:** Why are the `helloworld_init` and `helloworld_exit` functions declared `static`?

**Answer:** We want these functions to be private to this module and not be accessible to anywhere else, hence we make them `static`.

And you might have already guessed that the `printk` function is responsible for writing the strings in double quotes to the kernel log buffer.

Now that we know what the module does, lets move on to the next question, how do we generate the kernel object (`.ko`) file?

For, that lets move our discussion towards `Makefile` that I wrote to compile this module.

## The Makefile

Lets first take a look at the makefile and then we will go through what the lines in it do.

```makefile
PWD := $(shell pwd)
obj-m += helloworld.o

all:
			make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
clean:
			make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

So, the first thing to note is that we are using Kernel's `kbuild` system, which uses two variables `obj-y` and `obj-m`. 

- `obj-y` variable is a string that has concatenated list of all objects to build and merge into the final kernel image files — the `vmlinux` and `bzImage` files.
- `obj-m` variable is also a string that has concatenated list of all kernel objects to build separately as kernel modules. This is the exact reason why we have this line:
`obj-m += helloworld.o` in our makefile; we want to append this kernel object as well to that list.

Next, lets discuss the following segment in our makefile:

```makefile
all:
			make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
```

Here, the `-C` helps the make process to change the directory to the directory name that follows it. Hence, we move to the directory to the kernel `build` folder. This is done because of a rule in the Linux kernel, ***we can only insert a module into the kernel memory if that kernel module has been built against it*** — the precise kernel version, build flags, and even the kernel configuration options matter! 

This way, it is guaranteed that all kernel modules are tightly coupled to the kernel that they are built against with the exact same set of rules, that is, the compiler/linker configurations, as the kernel itself.

Next, there is an initialization of the variable named `M` and the target specified is `modules`; hence we move back to the directory specified by the `M` variable, which is the very folder where we started from.

Finally, there is following section:

```makefile
clean:
			make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

Now, when we build the module, few intermediate files are generated as well. So, in order to get rid of all those intermediate files and the kernel object file itself is the purpose of this target. When we run the command `make clean` it cleans up everything!

So, to summarize the building process:

- use `make` to build the kernel module object
- use `make clean` to clean up everything and start afresh!

Now, that we have written our module and compiled it, how do we see it in action?

So lets start from there!

## Loading the module in kernel memory

There is a command for it; and it is `insmod`. Just run the following command:

```bash
$ sudo insmod <path to your kernel object file>
```

Now, how do we verify if the module has been loaded in the kernel memory? If our module is correctly written, it should have printed "Hello, world" in kernel log buffer. But, how do we see the kernel log buffer's content? We can use a utility called `dmesg`. It prints the entire kernel log buffer to stdout, but since we are only interested in the last two lines of the buffer, so use the following command:

```bash
$ sudo dmesg | tail -n2
```

and this is the output from my machine

![Untitled](https://raw.githubusercontent.com/rast-7/blog/2063e8c1dbe573d10e212a55154052762d75faa9/content/articles/The%20First%20Kernel%20Module%20that%20I%20wrote%2046ae6d53b6c047da9b865306bfc8a195/Untitled%201.png)

## Removing the module from the kernel memory

Just use the `rmmod` command!

```bash
$ sudo rmmod <name of your module>
```

and just like before, we will use `dmesg` to check the whether our module really printed "Goodbye world" to the kernel log buffer before being removed from the memory.

```bash
sudo dmesg | tail -n1
```

![Untitled](https://raw.githubusercontent.com/rast-7/blog/2063e8c1dbe573d10e212a55154052762d75faa9/content/articles/The%20First%20Kernel%20Module%20that%20I%20wrote%2046ae6d53b6c047da9b865306bfc8a195/Untitled%202.png)

and it did!!

So, that is how we write a module, build it and use it!

I hope I was able to describe this process clearly, please let me know if you have any improvements or suggestions to improve the blog.

Thanks a lot for reading! 🙂