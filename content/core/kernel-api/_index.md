---
title: "Kernel API"
weight: 15
---

EVL heavily relies on [Dovetail]({{% relref "dovetail/_index.md" %}})
for its seamless integration into the main kernel. Among other things,
the latter extends the set of file operations which can be supported
by character drivers, so that they can handle I/O operations from the
out-of-band stage, while retaining the common structure of a character
device driver.

> Excerpt from _include/linux/fs.h_
```
struct file_operations {
	...
	ssize_t (*oob_read) (struct file *, char __user *, size_t);
	ssize_t (*oob_write) (struct file *, const char __user *, size_t);
	long (*oob_ioctl) (struct file *, unsigned int, unsigned long);
	__poll_t (*oob_poll) (struct file *, struct oob_poll_wait *);
	...
} __randomize_layout;
```

EVL drivers are common character device drivers, which only need to
implement one or more out-of-band handlers for performing real-time
I/O operations upon request from the EVL core.

In other words, there is no such thing as an _EVL driver model_, because
EVL fits into the regular Linux driver model.
