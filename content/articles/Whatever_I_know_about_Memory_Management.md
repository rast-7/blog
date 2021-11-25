---
title: "Whatever I know about Memory Management!"
date: "2021-10-19"
type: "article"
---

So, remember that when a `printf` function call happens, we eventually make a `write` system call to write to the `stdout`. When a system call happens, we move from the user space to the kernel space — and as we know, for every thread alive there are two stacks for it, a user space and a kernel space. But, following is the process we have seen till now:

![https://raw.githubusercontent.com/rast-7/blog/master/content/articles/My%20understanding%20of%20Processes%20and%20Threads!%20d9844200efdc4b9c84afb88791d57726/Untitled.png](https://raw.githubusercontent.com/rast-7/blog/master/content/articles/My%20understanding%20of%20Processes%20and%20Threads!%20d9844200efdc4b9c84afb88791d57726/Untitled.png)

and this uses all the memory available. So, how do we exactly call the `write` system call and where is the kernel stack that we discussed.

Now to answer that question, I will tell you that the image above is only two third of the actual image! The remaining one third of the image is the kernel address space!

Imagine, a 32-bit system, the total amount of virtual address space available to a process is 4GB, but that whole is split into some `user : kernel` ratio. This is called ***VM Split**.* So, for a 3:1 VM split configured system, out of the available 4GB virtual address space, 3GB is used by the process and the remaining 1GB is used by the kernel (called as kernel virtual address space).

On 64-bit systems, the available virtual address space is 2^64 bytes, which is 16EB. Any system doesn't have that much RAM so we don't use all of the 64-bits for addressing, instead we just use the first 48-bits. And, for the addresses of kernel virtual address space the MSB 16-bits are all set to 1 and for the addresses of the user virtual address space the MSB 16-bits are all set to 0. So, all the kernel virtual addresses follow the format `0xffff .... .... ....` and all the user virtual addresses follow the format `0x0000 .... ..... ....`. Also, now thinking about the 128:128 VM Split on 64-bit system, the processes have 128TB of address space in userland; and 128TB space for the kernel address space.

Now, lets get to know how can we examine the user virtual address space and the kernel virtual address space.

In order to get the user virtual address space information, just get the `pid` of the process and go through the contents of the file `proc/<pid>/map` . This exposes all the segments that we saw in the process anatomy picture.

Now, lets start looking at the kernel virtual address space! But before we move any further it is very important to undestand that all processes have their unique user virtual address space but they all share the same kernel virtual address space.

## The Kernel Space

The kernel space has a lot of things which are arch dependent, but there are a few things which are present everywhere, so lets take a look at them.

**The Lowmem Region —** This is where RAM is directly-mapped into the kernel. During the boot, the kernel maps all RAM directly into the kernel segment. So, we have following:

- Physical page frame 0 maps to kernel virtual page 0
- Physical page frame 1 maps to kernel virtual page 1

But why do we need this? The kernel maps  all the available memory before the applications start up, so that by the time the applications come up, kernel is ready for memory allocation. 

The kernel text, data and BSS segments also reside in the lowmem region.

**The kernel vmalloc region —** Core kernel or device drivers code can allocate virtually contiguous memory from this region using the `vmalloc()` API.

**The kernel modules space —** A region of the kernel VAS is set aside for memory taken up by the static text and data of the Loadable Kernel Modules.

**The Highmem region —** Suppose on a 32-bit system, we have 2GB of RAM, all of that RAM cannot be possibly mapped into the lowmem region at once, so the kernel sets up a region where it can set up a temporary virtual mapping of the remaining unmapped RAM. This is the highmem region.

This was present in the 32-bit systems, with 64-bit systems, since we have a very large amount of kernel virtual address space available, we don't need this region.

So, yeah that's what I know about the memory management till now

I am further exploring how the RAM is managed by the kernel and how the memory allocation happens in the kernel. I will write about that in a future blog. Stay tuned for that!

Please let me know if I was wrong somewhere in the blog, I would love to learn and improve!

Thanks a ton for reading! 🤓