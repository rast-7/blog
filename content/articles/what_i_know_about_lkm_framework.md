---
title: "What I know about the LKM Framework"
date: "2021-10-10"
type: "article"
---

Linux Kernel is a monolithic kernel. What I mean by this statement is that all the kernel components live in and share the kernel address space, and this address space is the virtual address space.

Now, lets imagine that we want to write a new device driver for a certain hardware chip. One way to do this is to update the kernel source tree with the new code, build it and then test it. This might seem like an intuitive thing to do, but this is a lot of work — every change in the code will require us to rebuild the kernel and then we will have to reboot the system in order to test it. So, the easier way to solve this problem is using *the LKM Framework*!

Oh and in case if you are wondering, the full form of LKM is Loadable Kernel Module!

The LKM framework is a means to compile a piece of kernel code outside of the kernel source tree, and plug it into the kernel memory, have it run and perform its job, and then unplug it from the kernel memory. Since it is kernel code and is running in the kernel memory, it runs with kernel privileges. This way, we don't have to reconfigure, rebuild and reboot the system each time. All we have to do is edit the code of kernel module, rebuild it, remove the old copy from the kernel memory and insert the new version.

The kernel module source code is typically written in C and uses Makefile to compile it. After compilation, we get a `.ko` (kernel object) file which is plugged into the kernel memory.

**Question:** How is this plugged into the kernel memory? 

**Answer:** By using the `insmod` command (also, I think I want to explore this more, so I will write a separate blog on how this actually plugs the module in the kernel memory)

**Question:** How is this unplugged from the kernel memory?

**Answer:** By using the `rmmod` command

**Question:** How do I see the kernel modules currently installed in my system?

**Answer:** Use the `lsmod` command

I think that was all that I knew about LKM!
In my next blog, we will explore how to write a kernel module whose purpose in life is to just say "Hello!" when plugged in to the kernel memory and "Goodbye!" when removed from the kernel memory!

Thanks a lot for reading! 🙂