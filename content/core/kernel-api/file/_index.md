---
title: "File description"
weight: 2
---

An [OOB-capable driver]({{< relref
"core/kernel-api/_index.md#evl-driver-definiton" >}}) is a device
driver which implements low latency operations on files like
[oob_read()]({{< relref "core/user-api/io/_index.md#oob_read" >}}),
[oob_write()]({{< relref "core/user-api/io/_index.md#oob_write" >}}) and
[oob_ioctl()]({{< relref "core/user-api/io/_index.md#oob_ioctl" >}}), in
addition to a set of common in-band operations such as
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html),
[close(2)](http://man7.org/linux/man-pages/man2/close.2.html),
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html),
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html),
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) etc.
[Out-of-band I/O requests]({{< relref "core/user-api/io" >}}) are
issued by [EVL threads]({{< relref "core/user-api/thread" >}}) running
in user-space. Those requests are directly handled from the
[out-of-band execution stage]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}), ensuring low
and bounded latency.

EVL files are kernel file objects (`struct file`) which have been
advertised by an OOB-capable driver as supporting these [out-of-band
I/O operations]({{< relref "core/user-api/io" >}}), in addition to
common in-band requests. For instance, [oob_ioctl()]({{< relref
"core/user-api/io/_index.md#oob_ioctl" >}}) requests may be issued for an EVL
file, like
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) may as
well. Such advertisement usually happens in the `open()` operation
handler of a character device driver by calling [evl_open_file()]({{<
relref "#evl_open_file" >}}) for the new file description being
initialized. Drivers which create (pseudo-)file objects with
out-of-band support on-the-fly from their
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) handler
may call [evl_open_file()]({{< relref "#evl_open_file" >}}) from this
context as necessary.

{{% notice tip %}}
A typical example of the latter usage can be found in the EVL-enabled [GPIOLIB
driver](https://git.evlproject.org/linux-evl.git/tree/drivers/gpio/gpiolib.c?h=evl/master),
which advertises out-of-band support for files created by the
GPIO_GET_LINEEVENT_IOCTL request from the [character device interface
to GPIO
lines](https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git). A
more usual way of advertising EVL files can be seen from the
`latmus_open()` routine of the [latmus driver
code](https://git.evlproject.org/linux-evl.git/tree/drivers/evl/latmus.c?h=evl/master)
which supports the [latmus application]({{< relref
"core/testing.md#latmus-program" >}})..
{{% /notice %}}

### Device file management services

---

{{< proto evl_open_file >}}
int evl_open_file(struct evl_file *efilp, struct file *filp)
{{< /proto >}}

Advertise _filp_ as a file which can be subject to out-of-band
handling. _filp_ is a pointer to a common file description (`struct
file`) usually received by the `open()` operation handler of a
character device driver. [evl_open_file()]({{< relref "#evl_open_file"
>}}) extends this in-band description with an out-of-band context
stored at _efilp_. Once [evl_open_file()]({{< relref "#evl_open_file"
>}}) has returned, applications can send [out-of-band I/O
requests]({{< relref "core/user-api/io" >}}) to this driver via any
file descriptor which refers to _filp_. [evl_open_file()]({{< relref
"#evl_open_file" >}}) does not block, it merely initializes _efilp_,
binds it to _filp_ then returns.

{{% notice tip %}}
_*efilp_ is usually part of the private data structure a driver
associates with each active file, as illustrated by the following
example:
{{% /notice %}}

```
#include <slab.h>
#include <evl/file.h>

struct foo_private_state {
       /* ... */
       struct evl_file efile;
};

static int foo_open(struct inode *inode, struct file *filp)
{
	struct foo_private_state *p;
	int ret;

	p = kzalloc(sizeof(*p), GFP_KERNEL);
	if (p == NULL)
		return -ENOMEM;
	
	ret = evl_open_file(&p->efile, filp);
	if (ret) {
	   	 kfree(p);
		 return ret;		 
	}

	filp->private_data = p;

	return 0;
}
```

{{% argument efilp %}}
A pointer to an EVL file context (`struct evl_file`), which is
initialized by this call. This memory object must live as long as
_filp_ is valid.
{{% /argument %}}

{{% argument filp %}}
The pointer to the file description structure received from the
[VFS](https://www.kernel.org/doc/Documentation/filesystems/vfs.txt) by
the `open()` operation handler of a character device driver.
{{% /argument %}}

[evl_open_file()]({{< relref "#evl_open_file" >}}) always succeeds,
returning zero in the current implementation. However, you may still
want to check the return code in case error conditions surface in a
later revision of this code.

---

{{< proto evl_release_file >}}
void evl_release_file(struct evl_file *efilp)
{{< /proto >}}

This is the converse operation to [evl_open_file()]({{< relref
"#evl_open_file" >}}), which _must_ be called by the `release()`
handler of an OOB-capable character device driver.

Because the in-band kernel and the EVL core run almost fully
asynchronously, some [out-of-band I/O operations]({{< relref
"core/user-api/io" >}}) - which the in-band kernel does not know about -
might still be pending for _efilp_ when the `release()` handler is
invoked by the in-band kernel. For instance, some application which
issued an [oob_read()]({{< relref "core/user-api/io/_index.md#oob_read"
>}}) request might still be waiting for data in the `oob_read()`
operation handler of the driver, sleeping on an EVL wait queue or a
[semaphore]({{< relref "core/kernel-api/semaphore/_index.md" >}}),
when some other thread of the program
[closes](http://man7.org/linux/man-pages/man2/close.2.html) all the
file descriptors referring to that file, leading to the `release()`
handler to be called by the
[VFS](https://www.kernel.org/doc/Documentation/filesystems/vfs.txt). In
order to track those operations, EVL maintains an internal reference
count on every device file registered with [evl_open_file()]({{<
relref "#evl_open_file" >}}). [evl_release_file()]({{< relref
"#evl_release_file" >}}) waits for this reference count to reach zero
before returning to the caller, acting as a synchronization barrier
with respect to any concurrent out-of-band operation which might be
running on any CPU. When this routine returns, you can be sure that no
such operation is ongoing in the system for the released file,
therefore it can be safely dismantled afterwards.

For this reason, you do want to call [evl_release_file()]({{< relref
"#evl_release_file" >}}) _before_ the context data attached to the
file being released is freed by the `release()` handler eventually
. The following example illustrates such precaution, considering that
some out-of-band operations might still be in-flight while the device
file is released by the in-band kernel, such as waiting on an [EVL
semaphore]({{< relref "core/kernel-api/semaphore/_index.md" >}}).

```
#include <slab.h>
#include <evl/flag.h>
#include <evl/file.h>

struct foo_private_state {
       /* ... */
       struct evl_flag eflag;
       struct evl_file efile;
};

static int foo_open(struct inode *inode, struct file *filp)
{
	struct foo_private_state *p;
	int ret;

	p = kzalloc(sizeof(*p), GFP_KERNEL);
	if (p == NULL)
		return -ENOMEM;
	
	ret = evl_open_file(&p->efile, filp);
	if (ret) {
	   	 kfree(p);
		 return ret;		 
	}

	evl_init_flag(&p->eflag);
	filp->private_data = p;

	return 0;
}

static int foo_release(struct inode *inode, struct file *filp)
{
	struct foo_private_state *p = filp->private_data;

	/*
	 * Flush and destroy the flag some out-of-band operation(s)
	 * might still be pending on. Some EVL thread(s) might be
	 * unblocked from oob_read() as a result of this call.
	 */
	evl_destroy_flag(&p->eflag);

	/*
	 * Synchronize with these operations. Unblocked threads will drop
	 * any pending reference on the file being released, eventually
	 * allowing this call to return.
	 */
	evl_release_file(&p->efile);

	/*
	 * Neither in-band nor out-of-band users of this file anymore
	 * for sure, we can free the per-file private context.
	 */
	kfree(p);

	return 0;
}
```

{{% argument efilp %}}
A pointer to the EVL file context (`struct evl_file`), which was
initialized by a previous call to [evl_open_file()]({{< relref
"#evl_open_file" >}}).
{{% /argument %}}

[evl_release_file()]({{< relref "#evl_release_file" >}}) may block
before releasing the file until all [references]({{< relref
"#evl_get_file" >}}) to _efilp_ are gone, which indicates that no
out-of-band operation is undergoing for this EVL file.

---

{{< proto evl_get_file >}}
struct evl_file *evl_get_file(unsigned int fd)
{{< /proto >}}

Search for the EVL file indexed on _fd_ and get a reference on it
atomically, which prevents its release by [evl_release_file()]({{<
relref "#evl_release_file" >}}) until the last reference is dropped
by a call to [evl_put_file()]({{< relref "#evl_put_file" >}}).

Internally, the EVL core holds a reference on any file subject to an
[out-of-band request]({{< relref "core/user-api/io" >}}) across the
call to the corresponding operation handler in the driver, so that
such file does not go stale unexpectedly while the request is still
ongoing. You may want to get additional references on EVL files from
any EVL driver which receives file descriptors directly from
applications via
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html)
arguments. Such reference should be held as long as the operation on
the underlying EVL file is not complete.

{{% notice tip %}}

The implementation of the [polling services]({{< relref
"core/user-api/poll/_index.md" >}}) in the [EVL
core](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/poll.c?h=evl/master)
illustrates how [evl_get_file()]({{< relref "#evl_get_file" >}}) is used
for retrieving EVL file descriptions from descriptors passed by the
application.

{{% /notice %}}

{{% argument fd %}}
A valid file descriptor referring to the EVL file.
{{% /argument %}}

This call returns a valid pointer to an active EVL file - such as
_efilp_ passed to [evl_open_file()]({{< relref "#evl_open_file" >}}))
- holding a reference on such file in the same move.

---

{{< proto evl_get_fileref >}}
void evl_get_fileref(struct evl_file *efilp)
{{< /proto >}}

Get a reference on the EVL file _efilp_. This is the core helper which
actually gets a reference on any EVL file. Once it has returned, the
file cannot be released by [evl_release_file()]({{< relref
"#evl_release_file" >}}) until the reference is dropped by a converse
call to [evl_put_file()]({{< relref "#evl_put_file" >}}).

{{% argument efilp %}}
A pointer to a valid EVL file description. This description was
originally initialized by a call to [evl_open_file()]({{< relref
"#evl_open_file" >}}).
{{% /argument %}}

---

{{< proto evl_put_file >}}
void evl_put_file(struct evl_file *efilp)
{{< /proto >}}

Drop a reference on an EVL file previously obtained by a call to
[evl_get_fileref()]({{< relref "#evl_get_fileref" >}}) or
[evl_get_file()]({{< relref "#evl_get_file" >}}).

If the reference count is zero at the time [evl_release_file()]({{<
relref "#evl_release_file" >}}) is called for _efilp_, the release is
performed immediately.

---

{{<lastmodified>}}
