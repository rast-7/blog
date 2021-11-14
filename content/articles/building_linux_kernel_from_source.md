---
title: "Building the Linux Kernel from source"
date: "2021-10-07"
type: "article"
---

The aim of this blog is to describe the process of building the Linux Kernel from source. There are certain steps involved in the process, and they are:

- Obtain the source code
- configure the Linux Kernel
- build the kernel image and modules
- installing the built kernel modules
- setup the GRUB bootloader and initramfs image
- customizing the GRUB bootloader menu

## Step 1 — Obtaining the Linux Kernel source

There are two ways in which we can obtain the Linux Kernel source code:

1. By downloading and extracting a specific kernel source code from [https://www.kernel.org](https://www.kernel.org) 
2. By cloning from various source trees. For example, the mainline kernel source tree (or Linus Torvald's source tree) or the linux-next tree (which has all the latest patches)

Let's start by cloning from the Linus Torvald's source tree. We can find it here: [kernel/git/torvalds/linux.git - Linux kernel source tree](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)

Now, in order to clone it, use the following command:
`git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git`

Since the Linux Kernel is a huge source code, it will take sometime to clone the source code depending on your internet speed.

## Step 2 — Configuring the Linux Kernel

Linux Kernel runs on a tiny embedded device to an enterprise class server, and they all share the same Linux Kernel source. They differ in just the configurations. Hence, configuring the Linux Kernel is the most critical step in the build process.

The infrastructure that the Linux Kernel uses to configure and build the kernel is known as the **kbuild** system. Some major components of this system are:

- **CONFIG_FOO —** Any configurable entitiy in kernel is represented by a **CONFIG_FOO** macro. Depending on the user's choice, the macro will resolve to one of `y`, `n` or `m`. `y` and `n` implies to yes and no, implying to build or not to build the feature into the kernel image itself. `m` stands for module and it implies to build the feature as a separate object, a kernel module.
- `Kconfig` **files —** These are the files where **CONFIG_FOO** symbols are defined.
- **Makefiles** — The kbuild system uses a recursive Makefile approach. The Makefile under the kernel source tree root folder is called the top-level Makefile, with a Makefile within each sub-folder to build the source there.
- `.config` **file** — This is the file where actual kernel configuration is written to. It is auto generated in the kernel's root folder. We should keep the working configs safe and also we should not directly edit this file, instead we should rely on tools like menuconfig for that.

So, how do we generate the `.config` file? 

There are serveral techniques, and some of those are:

- Use the existing distribution's kernel configuration — In this approach, we simply copy the existing Linux distribution's config file into the .config file in the root of our linux kernel source tree. Just use the following command:
`cp /boot/config-$(uname -r) .config`
- Build a custom configuration based on the kernel modules currently loaded in the memory — In this approach we provide the kbuild system with a snapshot of the kernel modules currently running on the system by simply redirecting the output of `lsmod` into a temp file, and then providing that file to the build. This can be achieved by following: 
`lsmod > /tmp/lsmod.now
cd <root of your cloned kernel>
make LSMOD=/tmp/lsmod.now localmodconfig`

Now, once we run the `make ... localmodconfig` there will be a difference in the configuration options between the kernel you are currently building and the kernel you are currently running the build on. In that case, the kbuild system will display every single config option and the available values you can set it to, on the terminal. Then, it will prompt you to select the value of any new config options it encounters in the kernel being built. Jere, we will take the easy way out: just press the `[Enter]` key to accept the default selection. 

After pressing the `[Enter]` key many times, the interrogation finishes and the kbuild system writes the newly generated configurations to `.config` file.

## Step 3 — Building the kernel image and modules

Just ensure that you are in the root of the configure kernel source tree and type `make`

That's it! The kernel image and any kernel modules will get built. This process will take time, so go get that much needed nap!

Now, after this completes, we will have two important files generated for us:

- **vmlinux —** it is the uncompressed kernel image
- **System.map —** the symbol-address mapping file
- **bzImage —** it is the compressed kernel image, the one that bootloader will actually load into the RAM, uncompress in memory and boot into. It can be found in `arch/x86/boot/`

## Step 4 — Installing the kernel modules

When we built the kernel image and modules, all the kernel config options which were marked as `m` have actually been built now. You can go and check in the respective folders, you will find a `*.ko` file (the kernel object file). 

Now, just having them generated is not enough, they need to be installed into a well known location within the root filesystem so that, at boot, the system can find and load them in the kernel memory. This is why we need to install the kernel modules, and that well-known location is  `/lib/modules/$(uname -r)`

The command that installs the modules is:
`sudo make modules_install`

## Step 5 — Generating the initramfs image and bootloader setup

This is a 2 step process, but there is a command that perform both the tasks, and that command is:
`sudo make install`

## Step 6 — Customizing GRUB

Before customizing, let's just keep a backup of original GRUB bootloader config file:
`sudo cp /etc/default/grub /etc/default/grub.original`

Now, lets edit it! We can choose an editor of our choice (which is vim 😛)
`sudo vim /etc/default/grub`

Now, to always show the GRUB prompt at boot, insert this line:
`GRUB_HIDDEN_TIMEOUT_QUIET=false`

Now, lets set the timeout to boot the default OS (in seconds) as required;

`GRUB_TIMEOUT=15`

Furthermore, if a `GRUB_HIDDEN_TIMEOUT` directive is present, just comment it out!

Finally, run `sudo update-grub` to have your changes take effect!

Now, that's it! We are done!! A new kernel, along with requested kernel modules and the initramfs image, have been generated, and the GRUB bootloader has been updated. Now, all that remains is to reboot the system, select the new kernel image on boot, boot up, log in and verify that all is well!