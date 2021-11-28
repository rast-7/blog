---
title: "Whatever Little I know about the Linux Device Model!"
date: "2021-11-01"
type: "article"
---

The first ever driver that I wrote was a misc char driver. It supported read and write functions and showed up in `dev/rastonit`. When the device node was written to, the data sent was checked and if it didn't match with my name, it returned an error. I had written it out of kernel source tree, as a loadable kernel module, so the device was was registered when the module was loaded and the device was unregistered when the module was unloaded.

## What is a misc char device?

If a device is not storage, nor a network device, then it's a character device. These devices cannot be mounted. A huge number of devices fall into the char device calls, including sensor chips, touch screens, camera, keyboards, mouse, and so on. 

Alright, as we know that the devices are identified using the major and minor numbers. In the kernel, `{major : minor}` pair is a single unsigned 32-bit quantity within the inode. Of these 32-bits, the MSB 12 bits represent the major number and the remaining LSB 20 bits represent the minor number. So, there can be around 4096 class of devices and each class of device can have around 1 Million minor number.

So, the device type and the `{major#, minor#}` pair are called as namespace. A common problem is that of the namespace getting exhausted. So, it was decided to collect various miscellaneous character devices into one class called the `misc` class, which is assigned the major number `10`.

So, the misc char device that we are going to write will have a major number of `10` and we will get a minor number dynamically allocated by the kernel. Lets get started with some code.  

## Section 1 — the headers to include

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/string.h>
#include <linux/fs.h>
#include <linux/miscdevice.h>
```

Notice the last 2 header files. As kernel driver creators, one of our job is to register our driver with appropriate Linux Kernel's framework; in out case, with the `misc` framework. This is done by using the `misc_register()` API. Which is defined in `miscdevice.h` . 

The Linux Device Model makes our life as device driver authors very easy. Since we are creating a device driver, there will be IO on the device file, which implies the user space threads can issue some system calls like `read`, `write`, etc on our device file. So, how do we take care of that? The good news is that the kernel takes care of that! We just have to define the functions which should be invoked when a user space process issues the system calls on our device; and when we register our device to the `misc` framework we pass this information then, so that the kernel can invoke the required functions. 

We store all this information about which function to invoke when certain system call is made in a structure called `file_operations` which is defined in `linux/fs.h` and that is why we have to include that header as well.

## Section 2 — The file_operations struct

The `file_operations` structure is of critical importance! The majority of the members of this structure are function pointers, which point to functions which define the possible file-related system calls that could be issued on a device file. So, it has `open`, `read`, `write`, `poll`, `mmap` and several more members. So our job here is to populate with these function pointers with the functions that are supported by our device. We are going to support only the `open` and `read` operations. So the `file_operations` structure looks like following in our case:

```c
static const struct file_operations misc_fops = {
		.read  = read,
		.write = write,
};
```

So, when from the user space some process reads your device's file, by issuing the `read` system call on it. The kernel virtual file system will redirect the control to the device driver, and the `read` function that you defined will be invoked. So, our job as the device driver author is to provide that functionality. So, we will have to create two functions called `read` and `write`.

## Section 3 — The miscdevice struct

```c
static struct miscdevice miscdev = {
		.minor = MISC_DYNAMIC_MINOR,
		.name  = "rastonit",
		.mode  = 0666,
		.fops  = &misc_fops,
};
```

In the `miscdevice` structure, we did the following:

- We set the `minor` field to `MISC_DYNAMIC_MINOR` so that the kernel dynamically assign us an available minor number.
- We initialized the `name` field to `rastonit`. So, when the device is successfully registered, the kernel framework will automatically create a device node on our behalf in `/dev` of the form `/dev/rastonit`.
- We set the permissions of the device node to `0666` which means that our device node will be readable and writeable by all.
- We set the `fops` to a structure which we defined in section 2, which tells the kernel what all functions our driver is going to support. So, when a user space thread invokes those system calls on our device file, the kernel knows which functions to invoke in the device driver.

## Section 4 — The read function

```c
static ssize_t read(struct file *f, char __user *buf, size_t count, loff_t *offset)
{
	return simple_read_from_buffer(buf, count, offset, "rajat", strlen("rajat"));
}
```

First of all, how did I know the signature of the `read` function? As we discussed in section 2, in `file_operations` structure we map the system calls to function pointers which should be invoked, when that function call is called from the user space program. So, if you go ahead and take a look at the file `include/linux/fs.h` in the linux kernel source. You will find this:

![Untitled](The%20first%20ever%20driver%20that%20I%20wrote!%204f6982c819124ddd878d98d7a56517d0/Untitled.png)

So, this structure has members which are function pointers. If you take a look at the `read` field, you will notice that the definition is same what you see in this screenshot! That is how I got to know the definition of the `read` function.

So, what we are doing in this function is just copying the string, `"rajat"` from the kernel space to a buffer in user space. The API that we are using is called `simple_read_from_buffer`. This API helps us to transfer the string to the user space buffer securely!

## Section 4 — The write function

```c
static ssize_t write(struct file *f, const char __user *buf, size_t count, loff_t *offset)
{
	char msg[6] = {0};
	int ret;

	ret = simple_write_to_buffer(msg, sizeof(msg), offset, buf, count);
	if (ret < 0)
		return ret;

	if (!strncmp(msg, "rajat", strlen("rajat")) 
		&& count - 1 == strlen("rajat"))
			return count;
	
	return -EINVAL;
}
```

So, what we are doing here is that we are reading from a user space buffer to a buffer in the kernel space and using the `simple_write_to_buffer` API for that. Once we read the string, we check whether that string is "rajat", if it is, we return the number of bytes written and if it does not, we return an error code.

## Section 5 — The init function

Now, that we have defined the global structures, `file_operations` and `miscdevice`, have implemented the functions that our driver will support — `read` and `write`, lets go ahead and create the `_init` function.

```c
static int __init hello_world(void)
{
	int ret;

	ret = misc_register(&miscdev);
	if (ret)
		pr_info("Unable to register rastonit\n");
	else
		pr_info("misc device rastonit successfully registered.\n");
	
	return ret;
}
```

There is only one thing happening here, as soon as we load the kernel, we are registering our device to the `misc` framework using the `misc_register()`API. This API just takes one parameter, the pointer to `miscdevice` struct. If this function returns success, we have successfully registered our device to the `misc` framework and our device should be ready to use using this driver!

## Section 6 — The exit function

```c
static void __exit goodbye(void)
{
	pr_info("Goodbye!\n");
	misc_deregister(&miscdev);
}
```

So yeah, as expected! When the module is unloaded from the kernel memory, our device is also un-registered from the `misc` framework using the `misc_deregister` API. This API takes the same argument as `misc_register()`, the `miscdevice` struct. 

Those are all the sections our device driver, lets put them all together and see!

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/string.h>
#include <linux/fs.h>
#include <linux/miscdevice.h>

MODULE_LICENSE("GPL");

static ssize_t read(struct file *f, char __user *buf, size_t count, loff_t *offset)
{
	return simple_read_from_buffer(buf, count, offset, "rajat", strlen("rajat"));
}

static ssize_t write(struct file *f, const char __user *buf, size_t count, loff_t *offset)
{
	char msg[6] = {0};
	int ret;

	ret = simple_write_to_buffer(msg, sizeof(msg), offset, buf, count);
	if (ret < 0)
		return ret;

	if (!strncmp(msg, "rajat", strlen("rajat")) 
		&& count - 1 == strlen("rajat"))
			return count;
	
	return -EINVAL;
}

static const struct file_operations misc_fops = {
		.read  = read,
		.write = write,
};

static struct miscdevice miscdev = {
		.minor = MISC_DYNAMIC_MINOR,
		.name  = "rastonit",
		.mode  = 0666,
		.fops  = &misc_fops,
};

static int __init hello_world(void)
{
	int ret;

	ret = misc_register(&miscdev);
	if (ret)
		pr_info("Unable to register rastonit\n");
	else
		pr_info("misc device rastonit successfully registered.\n");
	
	return ret;
}

static void __exit goodbye(void)
{
	pr_info("Goodbye!\n");
	misc_deregister(&miscdev);
}

module_init(hello_world);
module_exit(goodbye);
```

In order to compile it, we will have to write a makefile.  The makefile that I used for this module is same as the one that I used for my hello, world module. It can be found here: [The first kernel module that I wrote | Ups and Downs (rajatasthana.com)](https://www.rajatasthana.com/articles/the_first_kernel_module_that_i_wrote/)

So, in order to test our device driver we will do two things:

- Once we load the module using the `insmod` we will check if the device can be found in the `/dev` directory as `/dev/rastonit`
- Next in order to test whether the `read` function, we will use `cat` system call
- And, in order to test the `write` function, we will use `echo` system call.

The following screenshot does it all!

![Untitled](The%20first%20ever%20driver%20that%20I%20wrote!%204f6982c819124ddd878d98d7a56517d0/Untitled%201.png)

Notice the output when we did `ls -l` in `/dev/rastonit`. Our device got registered as a char device. The proof that it is a device of `misc` class can be found by looking at the `major number` which is `10` here. The dynamically allocated `minor number` for our device was `119`.

When we do `cat` on our device, our implemented `read` function got invoked and it did as we asked it to — return the string `"rajat"` to the user space and it did! The `cat` printed it on stdout!

When we do `echo` and give it a string other than `"rajat"` we get an invalid argument error, as expected! And when we write `"rajat"` to our device file, the operation is succesful! The `echo $?` confirms that.

So, I hope this blog helped you and you might be feeling a bit better about your understanding of these things 🤞

Please let me know if there is anything that can be improved about this blog, I would love to get some feedback!

Thanks a ton for reading! 🤓