---
title: "Whatever Little I know about the Linux Device Model!"
date: "2021-10-25"
type: "article"
---

I know very little about the Linux Device Model, but whatever I am writing in this blog is whatever I kept in my mind and anyway got started writing my first device driver, a very very simple misc char device driver! I hope this helps anyone who is starting out...

With that disclaimer, lets get started!!

### What is a device driver?

A device driver is the interface between the OS and a hardware device. Basically, it is a piece of kernel code that lets the operating system interact with the device. Every device attached to your system has a device driver of its. It can be written in the kernel source tree itself, or outside the kernel source tree as a loadable kernel module.

I am pretty sure you must have heard the popular saying, *"Everything is a file!".* That applies even to the devices. If some user space program wants to interact with any device, it will have to open a special type of file — a device file! These files can be found in the `/dev` directory.

In the Linux Kernel, devices are organized within a tree-like hierarchy within the kernel. The hierarchy is first divided based on device type — block, char, socket, pipes, etc. Within that, we have some major numbers for each type, and each major number is further classisified in some minor numbers.

So, if you go to `/dev` directory on your system and execute `ls -l` command, it will list the devices connected to your system. This is a screenshot of a part of the output I got on my terminal...

![Untitled](Whatever%20little%20I%20know%20about%20the%20Linux%20Device%20Mode%202ccc6a993196409cba9957c34b3bb041/Untitled.png)

Please notice the first column, which shows the type of file and the permissions. The first character (the one underlined in red) represents the type of file. So, in this figure you are only seeing three file types, but there can be 7 types of files: 

- `l` — Represents a symbolic link (think of this as a shortcut to a different file or directory)
- `c` — Represents a char device
- `b` — Represents a block device
- `d` — Represents a normal directory
- `s` — Represents a socket
- `p` — Represents named pipes
- `-` — Represents normal files

So, this is one way kernel uses to distinguish between different file types. This information about the file type is stored in the file's inode data structure.

The other information that the kernel uses to distinguish different files is it's major and minor number. This information is also present in the file's inode data structure.

### What does this major and minor number really mean?

The major number represents a class of device. For example, a keyboard, joystick, sensor, touchscreen etc. 

The minor number's meaning is decided by the driver author. For example, a storage device deriver (which is a block device) can use minor number to represent different partitions in it.

 

We can use the following figure to keep these things in mind. Please note that I have only shown block and char devices, but there are more devices...

![Untitled Diagram-Page-4.svg](Whatever%20little%20I%20know%20about%20the%20Linux%20Device%20Mode%202ccc6a993196409cba9957c34b3bb041/Untitled_Diagram-Page-4.svg)

Now, with these basic things in mind, lets talk about the Linux Device Model!

### The Linux Device Model

The linux device model ties the following major components:

- The buses on the system
- The devices on the buses
- The device drivers that interacts with the devices

A basic principle is that every single device must reside on a bus. For example, USB device on USB bus, I2C devices on I2C bus and so on. You can take a look at the buses on the system by taking a look on the things present under the `sys/bus` directory on your system. You can use `sys` directory as a means to view things in the device model.

The kernel's driver core provides bus drivers, we don't have to worry about them, the kernel already has them.  The bus drivers organize and recognize the devices on them. Whenever a new device comes up, the bus driver will recognize it, it will then set up the device, allocates resources for it, registers it on whichever interface it has to and a lot more things!

The device driver has to take care of two things:

- The driver should register iteself to a kernel framework. For example, a driver for a network adapter should register itself to the kernel's network infrastructure. There are APIs to do these this.
- The driver should register itself to a bus. As in the previous example, if the network adapter wants to run on a PCI bus, then it should register to the PCI bus. There are APIs to do this as well.

Once these things are taken care of, the device will be identified by the kernel. Now lets say we have a storage device, and since its a storage device IO operations will be happening on it. So what about those? How does the user space applications interact with the device?

Communicating with the user space applications is taken care of by the kernel. As a driver creator you just have to provide functionality. For example, you have to implement `open`, `read`, `close`, `seek`, `poll` functions. I mean to say whatever functions are supported by your device. So, when the user space application wants to perform `open` system call on your device, the kernel will call the `open` function that you implemented in your driver. So, your life as a device driver has really been made very easy!

This is the whatever little I know about the LDM. In my next blog, I will walk you through the first driver that I wrote! 

Please let me know, if there is anything that I wrote is not correct, I would love to learn and improve!

Thanks a ton for reading! 🤓