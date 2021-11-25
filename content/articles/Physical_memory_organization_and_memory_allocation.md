---
title: "Physical memory organization and memory allocation and deallocation!"
date: "2021-10-25"
type: "article"
---

One of the best that has ever happened to me was that I got an opportunity to take part in the Eudyptula challenge! In that challenge, there are problems around memory allocation and deallocation, which made to explore how the kernel manages the physical memory, and the algorithms and APIs involved in the allocation and deallocation of memory. Lets get started!

# Physical Memory Organization

Lets begin by understanding some terms:

**Nodes —** These are data structures used to denote a physical RAM module on the system motherboard and its associated controller chipset.

**Zones —** A node is divided into zones

**Page Frames —** These are physical pages of RAM.

So at boot, the kernel organizes and partitions into a tree-like hierarchy consisting of nodes, zones and page frames. 

- Nodes are divided into zones
- Zones consist of page frames

## Buddy System Algorithm for memory allocation & deallocation

In buddy system algorithm, we maintain a metadata structure which is an array of pointers pointing to doubly linked circular lists. This size of the array is equal to `MAX_ORDER - 1`. The value of `MAX_ORDER` is arch-dependent. The index of the array of pointers is called order of the list — it is the power to which to raise 2 to. 

On x86 systems, `MAX_ORDER` is 11, so that means the order ranges from $2^0$ to 2^10. But what does that mean?

Each doubly linked circular list points to free physical contiguous page frames of size 2^order.. Thus, the elements in the first index of the array point to contiguous chunks of 4KB. and the the elements of the last index points to contiguous chunks of 4MB.

The kernel gives us a convenient view into the current stat of the page allocator via the `/proc` filesystem. Just check the output of `cat /proc/buddyinfo`. This is what the output looks like on my system: 

![Untitled](Physical%20memory%20organization%20and%20memory%20allocation%2093cee87d285e4fa694431edce6cb507c/Untitled.png)

This also tells a lot about my system, it has only single node, which means just one RAM. There are 3 zones: DMA, DMA32 and Normal. After that there are 10 columns. These 10 columns are nothing but the number of free (physically contiguous) page frames in order 0, order 1, right up to order 10.

So, on my system, there are 2 single-page free chunks of RAM in the order 0 list for node 0, zone DMA.

Let's understand how this page allocation works!

Lets take an example, we need 128KB of memory, the algorithm will do this:

1. It expresses the amount to be allocated in pages. So, here we have (128/4) = 32 pages.
2. Next, it determines to what power 2 must be raised to get 32. Which is 5.
3. Now, it checks the list on order 5 of the appropriate node:zone page allocator freelist. If a memory chunk is available, dequeue it from the list, update the list, and allocate it to the requester.
4. If no memory chunk is available on the order 5 list, then it checks the list on the next order; that is, the order-6 linked list. If the order-6 is not null, it will dequeue a chunk of memory from it (which is 256KB) and do the following:
- Update the list to reflect the fact that one chunk is removed
- Cut the chunk in half, so now we have 2 128KB halves
- Migrate one half to the order-5 list
- Allocate the other half to requester
5. If the order-6 list is also empty, then it repeats the preceeding process with the order-7 list, and so on, until it succeeds.
6. If all the remaining higher-order lists are empty, it will fail the request.

What happens when we free the 128KB memory that got from the order-6 list??

The algorithm calculates that the just-freed chunk belongs to order-5 list. But before blindly enqueuing it there, it looks for its buddy block, and in this case, it finds it! It merges the the two buddy blocks into a single larger block and places the merged block in the order-6 list. 

This helps in defragment the memory!

### The downfall case for the algorithm

If we request for a 132KB block. The algorithm allocates more than 132KB, it allocates 256KB. Since the customer requested for only 132KB, the remaining 124KB is wasted. This is called ***internal fragmentation***. 

## Lets Cache Things!

I read somewhere that some developer noticed certain kernel data structures were allocated and deallocated quite frequently within the OS. He thus had the idea of per-allocating them in a cache. This evolved into what we currently know as ***slab cache.***

Thus on linux as well, the kernel pre-allocates a fairly large number of objects into several slab caches for performant allocation of memory. The current state of the slab caches can be viewed by looking at the contents of `/proc/slabinfo` file.

The `kmalloc()` and `kfree()` APIs allocate and deallocate memory from the slab caches.

I won't be describing the details about the APIs as we can read about them from the documentation.

Please correct me if I was wrong anywhere in the blog! I would love to learn and improve!!

Thanks a ton for reading!!!! 🤓