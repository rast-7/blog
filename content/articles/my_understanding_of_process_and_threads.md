---
title: "My understanding of Process and Threads!"
date: "2021-10-15"
type: "article"
---

I really have a weak memory, and am very forgetful! So, I just want to write this down so that I can come back and look at this, or even correct some facts which I might have interpreted wrongly (on that note, if you find something that is not correct, please correct me!)

With that out of the way, lets get started!

# What is a Process and what is a thread?

A process is an active instance of a program. 

And a thread is merely an execution path within a process. I really like this definition of thread. This is just one of the definition that helped me understand the concept.

So, since thread is merely an execution path within a process, so that means, in all the C programs there is atleast 1 thread; and that one thread which is always there is our renowned `main` function!

And also whenever we talk about threads, we mostly are thinking of different things happening on different threads; that means they might be calling different functions. But, threads are part of process; so that means that every thread has its own stack; but since threads are part of the process they share every other thing. Which brings our next topic: what are these things that all the threads share.

# Anatomy of a process

Oh! how can we discuss anatomy without an image! So following is how a process looks!

![Untitled](My%20understanding%20of%20Processes%20and%20Threads!%20d9844200efdc4b9c84afb88791d57726/Untitled.png)

Well this is really a bad drawing! This proves, I cant even use paint 😢 But hope it conveys the idea.

Lets go through each section from the bottom:

- **Text Segment —** This is where machine code is stored
- **Initialized data segment —** This is the section where pre-initialized variables are stored
- **Uninitialized data segment —** Uninitialized variables are stores here. These values are auto-initialized to 0 at runtime; this region is called **bss**
- **Heap segment —** You might recognize this section from the C library function, malloc. This is the segment from where malloc gets the memory (but thats not fully correct, if the requested amount of the memory is less that a certain amount only then it gets memory from heap, else there are other mechanism through which it gets memory which I am not aware of at the moment). It grows from the lower address to higher address.
- **Library —** All libraries that are linked to the process are mapped here
- **Stack —** This is the section used for the purpose of implementing the function-calling mechanism. It grows from higher address to lower address.

And, as we know that threads are just an execution path within a process, I have shown three threads in the process figure above. Two zig-zag thingy in the text section in blue color represent thread1 and thread2; and the black zig-zag thingy represent the main() thread. In the stack section notice two blue blocks, they represent different stacks for each of thread1 and thread2; and a stack for the main() thread. So, I hope that conveys the message that the thread shares all process resources except for the stack. Each thread having their own stack helps then to run in parallel.

Ah! and the last line of the previous paragraph, Each thread having their own stack helps them to run in parallel, brings another point. ***Threads run, not processes.*** That is the thread is the kernel schedulable entity and not the process.

In Linux, every thread has a corresponding ***task structure**.* For every thread alive, the kernel maintains a task structure.

Now, lets move further. Have you ever thought how `printf` actually is able to print the string that we pass to it? 

Well what happens is that `printf` function is C library function which eventually calls a system call called `write` to write on the `stdout`. So, when a system call is made does the thread on which `printf` is running, requests another request in the kernel space to call the `write` system call on its behalf and return success when done? No, this does not happen, the thread which was running in the user space transitions in the kernel space and runs there and calls the `write` system call. This is called as running kernel code in process context.

Now, this brings another question, so now that the same thread went into the kernel space and executed, so when the `write` system call must have been called, the same stack which was being used when that thread was in user-space must have been used? The answer to that is nope! 

On Linux, two privilege levels are supported — ***the unprivileged user mode*** and ***the privileged kernel mode.*** Thus, every thread alive has two stacks — A user space stack and a kernel space stack.

The user space stack is in play when the thread executes in user-mode and the kernel space stack is in play when the thread switches to kernel mode (via a system call or interrupt) and executes kernel code paths.

### Can we look at these two stacks?

Yes! definitely we can!

In order to view the kernel stack for any process, just get the `pid` of the process and look up this file in `/proc`.  

`cat /proc/<pid>/stack`

And, to use the user space stack you will have to use GDB. Just run your process, attach gdb to it and use the `backtrace` command.

So! that is all I know about this! 

I hope it is easy to understand and helped you in some way!

Please let me know if I am wrong about something! and as always thanks a ton for reading!!! 🤓