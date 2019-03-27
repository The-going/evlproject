---
title: "Driver interface"
weight: 15
pre: "&rsaquo; "
---

EVL heavily relies on [Dovetail]({{% relref "dovetail/_index.md" %}})
for its seamless integration into the main kernel. Among other things,
the latter extends the set of file operations which can be supported
by file descriptions, so that they can handle I/O operations from the
out-of-band stage upon request from an autonomous core such as EVL,
while retaining the common structure of a character device
driver. Applications running in user-space can start these I/O
operations, by issuing specific system calls to EVL.

> Excerpt from _include/linux/fs.h_, as modified by Dovetail
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

Therefore, EVL drivers are common character device drivers, which only
need to implement some out-of-band handlers for performing real-time
I/O operations upon request from the EVL core. In other words, there
is no such thing as an _EVL driver model_, because EVL fits into the
regular Linux driver model.

Dovetail is in charge of routing the system calls received from
applications to the proper recipient kernel, either EVL or the main
one.  As mentioned earlier when describing the
[everything-is-a-file]({{% relref
"core/_index.md#everything-is-a-file" %}}) mantra EVL implements, only
I/O transfer and control requests have to run from the out-of-band
context (i.e. EVL's real-time mode), establishing and managing the
underlying file channel from the regular in-band context is just
fine. Therefore, the execution flow upon I/O system calls looks like
this:

![Alt text](/images/oob_calls.png "Out-of-band I/O handling")

Which translates as follows:

- When an application issues the `open(2)` or `close(2)` system call
  to an EVL driver, the request goes through the virtual filesystem
  switch (VFS) as usual, ending up into the `.open()` and `.release()`
  handlers (when the last reference is dropped) defined by the driver
  in its `struct file_operations` descriptor. The same goes for
  `mmap(2)`, `ioctl(2)`, `read(2)`, `write(2)`, and all other common
  file operations for character device drivers.

- When an applications issues a `oob_read()`, `oob_write()` or
  `oob_ioctl()` system call via the EVL library, Dovetail routes the
  call to the EVL core (instead of the VFS), which in turn fires the
  corresponding handlers defined by the driver's `struct
  file_operations` descriptor: i.e.  `.oob_read()`, `.oob_write()`,
  and `.oob_ioctl()`. Those handlers should use the EVL kernel API, or
  the main kernel services which are known to be usable from the
  out-of-band-context, _exclusively_. Failing to abide by this golden
  rule may lead to funky outcomes ranging from plain kernel crashes
  (lucky) to rampant memory corruption (unlucky).

{{% notice tip %}}
Now, you may wonder: _"what if an out-of-band operation is ongoing in
the driver on a particular file, while I'm closing the last reference
to that file?"_ Well, the `close(2)` call will block until the
out-of-band operation finishes, at which point the `.release()`
handler may proceed to dismantling the file. A [simple rule]
({{% relref "core/kernel-api/file/_index.md#evl_release_file" %}}) for
writing EVL drivers ensures this.
{{% /notice %}}
