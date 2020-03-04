---
title: "Writing drivers"
weight: 15
pre: "&rsaquo; "
---

## What is an EVL driver? {#evl-driver-definition}

An EVL driver is a regular Linux character device driver which also
implements a set of out-of-band I/O operations advertised by its file
operation descriptor (`struct file_operations`). These out-of-band I/O
requests are only available to [EVL threads]({{< relref
"core/user-api/thread" >}}), since only such threads may run on the
[out-of-band execution stage]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}). Applications
running in user-space can start these I/O operations by issuing
[specific system calls]({{< relref "core/user-api/io" >}}) to the EVL
core.

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

[Dovetail]({{% relref "dovetail/_index.md" %}}) is in charge of
routing the system calls received from applications to the proper
recipient, either the EVL core or the in-band kernel.  As mentioned
earlier when describing the [everything-is-a-file]({{% relref
"core/_index.md#everything-is-a-file" %}}) mantra, only I/O transfer
and control requests have to run from the out-of-band context
(i.e. EVL's real-time mode), creating and dismantling the underlying
file from the regular in-band context is just fine. Therefore, the
execution flow upon I/O system calls looks like this:

![Alt text](/images/oob_calls.png "Out-of-band I/O handling")

Which translates as follows:

- When an application issues the
  [open(2)](http://man7.org/linux/man-pages/man2/open.2.html) or
  [close(2)](http://man7.org/linux/man-pages/man2/close.2.html) system
  calls with a file descriptor referring to a file managed by an EVL
  driver, the request normally goes through the virtual filesystem
  switch (VFS) as usual, ending up into the `.open()` and `.release()`
  handlers (when the last reference is dropped) defined by the driver
  in its `struct file_operations` descriptor. The same goes for
  [mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html),
  [ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html),
  [read(2)](http://man7.org/linux/man-pages/man2/read.2.html),
  [write(2)](http://man7.org/linux/man-pages/man2/write.2.html), and
  all other common file operations for character device drivers.

- When an applications issues the [oob_read()]({{< relref
  "core/user-api/io/_index.md#oob_read" >}}), [oob_write()]({{< relref
  "core/user-api/io/_index.md#oob_write" >}}) or [oob_ioctl()]({{<
  relref "core/user-api/io/_index.md#oob_ioctl" >}}) system calls via
  the EVL library, Dovetail routes the call to the EVL core (instead
  of the VFS), which in turn fires the corresponding handlers defined
  by the driver's `struct file_operations` descriptor: i.e.
  `.oob_read()`, `.oob_write()`, and `.oob_ioctl()`. Those handlers
  should use the EVL kernel API, or the main kernel services which are
  known to be usable from the out-of-band-context,
  _exclusively_. Failing to abide by this golden rule may lead to
  funky outcomes ranging from plain kernel crashes (lucky) to rampant
  memory corruption (unlucky).

{{% notice tip %}}
Now, you may wonder: _"what if an out-of-band operation is ongoing in
the driver on a particular file, while I'm closing the last
VFS-maintained reference to that file?"_ Well, with a properly written
EVL driver, the
[close(2)](http://man7.org/linux/man-pages/man2/close.2.html) call
should block until the out-of-band operation finishes, at which point
the `.release()` handler may proceed with dismantling the file. A
[simple rule] ({{% relref
"core/kernel-api/file/_index.md#evl_release_file" %}}) for writing EVL
drivers ensures this.
{{% /notice %}}

---

{{<lastmodified>}}
