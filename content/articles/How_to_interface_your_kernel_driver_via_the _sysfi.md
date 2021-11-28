---
title: "How to interface your kernel module via the sysfs"
date: "2021-11-03"
type: "article"
---

In my previous blog, [The first ever driver that I wrote | Ups and Downs (rajatasthana.com)](https://www.rajatasthana.com/articles/the_first_ever_driver_that_i_wrote/), I had created a very simple misc char device driver. It interacted with the user space using the `simple_read_from_buffer()` and `simple_write_to_buffer()` APIs. That is one way to interact with the user space processes, but there are other and much better ways to do this. In this blog, lets explore how can we interface a kernel module using the sys filesystem!

## The sysfs!

The sysfs tree encompasses the following:

- Every bus present on the system
- Every device present on every bus
- Every device driver bound to a device on a bus

It provides different viewports into the Linux Device Model.

Using sysfs, we can view the system in several different ways; for example, we can view the system via the various buses it supports, via various classes of devices, via the devices themselves, and so on. 

Now, with that background lets get started and look at very integral component of sysfs — ***kobject.***

## Linux Kernel Objects

Folders in each sysfs directory are represented by kernel objects (`kobjects`) in Linux. These are represented in the kernel through the `struct kobject`. Some of the important members of the kernel objects are:

- `name` — This is the name with which the folders are created within specified `/sys` directory.
- `parent` — When a folder is created in sysfs, if this field is present, the the kobject folder is created inside the parent kobject folder.
- `kref` — This is a reference counting mechanism for kobject. Whenever a kernel module refers to a kobject, its reference count is incremented, and whenever any kobject reference is released by any kernel module, the reference count is decremented. When the reference count decrements to 0, the kobject is removed from the memory.

## Creating a directory in sysfs

The API that we are going to use for creating the directory within sysfs is `kobject_create_and_add()`. It takes two arguments:

- `name` — The name of the directory to be created. It is a char array.
- `parent` — This is the kobject pointer of the parent directory in which the current directory is to be created. For example, if a sysfs directory has to be created under `/sys/kernel` , the parent that will be used is `kernel_kobj`. Similarly, if a sysfs directory has to be created under `/sys/firmware` then the parent that will be used is `firmware_kobj`. 
And, if we want to create the folder in `/sys` then the parent pointer is kept as `NULL`.

## Creating files in sysfs

Now that we know how we can create a directory in sysfs, lets move further in our discussion and see how can we expose information by creating a file in sysfs. 

Lets say, our driver has some attribute `example_attribute` which we want to expose to user space using sysfs, then the feature that we will be using to do so is called `sysfs attributes`. Attributes are of type `struct attribute`. It has the following fields:

- `name` — The name of the attribute to be created
- `mode` — The permission with which the file is to be created

Now, this `struct attribute` is embedded in `struct kobj_attribute` as follows:

```c
struct kobj_attribute {
	struct attribute attr;
	ssize_t (*show) (struct kobject *kobj, struct kobj_attribute *attr, char *buf);
	ssize_t (*store) (struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count);
};
```

here, `attr` is the attribute representing the file to be created, `show` is the pointer to the function that will be called when the file is read in sysfs, and `store` is the pointer to the function which will be called when the file is written in sysfs.

Once we have all the attributes of the `kobj_attribute` we can use a macro called 

`__ATTR(name, permission, show_ptr, store_ptr)` to create a `kobj_attribute` 

Then the `kobj_attribute` that we created just now, is grouped in the `struct attribute` group as follows:

```c
struct attribute *example_attr[] = {
	&example_attr.attr,
	NULL,
};
```

Then the above created attribute is given to the attribute group as follows:

```c
struct attribute_group example_attr_group = {
	.attrs = example_attr,
};
```

Finally, once the `attribute_group` object is ready, we can use the `sysfs_create_group()`API to create files under the sysfs. This API takes two arguments, one is the pointer to kobject of the directory under which the file has to be created and second parameter is the `attribute_group` that we just created.

Now that we have all the pieces of the puzzle, lets put the puzzle together!

Following is simple loadable kernel module, which interfaces with the user space via sysfs!

```c

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/string.h>

MODULE_LICENSE("GPL");

static const char *MYNAME = "RAJAT";

static struct kobject *dir;

static ssize_t id_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf) {
	return sysfs_emit(buf, "%s\n", MYNAME);
}

static ssize_t id_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count) {
	if (count - 1 == strlen(MYNAME) && !strncmp(buf, MYNAME, strlen(MYNAME)))
		return count;

	return -EINVAL;
}

static struct kobj_attribute id_attribute = __ATTR_RW(id);

static struct attribute *attrs[] = {
	&id_attribute.attr,
	NULL,
};

static struct attribute_group attr_group = {
	.attrs = attrs,
};

static int __init hello_world(void) {
	int ret;
	dir = kobject_create_and_add("rastonit", kernel_kobj);
	if (!dir) {
		pr_debug("Failed to created /sys/kernel/rastonit.\n");
		return -ENOMEM;
	}

	ret = sysfs_create_group(dir, &attr_group);
	if (ret)
		kobject_put(dir);

	return ret;
}

static void __exit goodbye(void) {
	pr_info("Goodbye! \n");
	kobject_put(dir);
}

module_init(hello_world);
module_exit(goodbye);
```

Now, all that remains is to write a Makefile, compile load and test the module!

I am trusting you to write the Makefile 👊

We will verify be checking if the file named `id` exists inside a folder named `sys/kernel/rastonit`. Here is what is looks like on my system:

![Untitled](https://raw.githubusercontent.com/rast-7/blog/master/content/articles/How%20to%20interface%20your%20kernel%20driver%20via%20the%20sys%20fi%2042bfaa1bbe5442009fb2a8b85c33427a/Untitled.png)

Notice that we do have a file called `id` under `/sys/kernel/rastonit` directory!

and on issuing `cat` it is returning `MYNAME`, which was expected.

So with this we come to end of this blog!

I hope this helped you learn something. Please let me know if this blog can be improved in any way!

Thanks a ton for reading 🤓