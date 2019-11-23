---
title: "Out-of-band I/O services"
menuTitle: "Out-of-band I/O"
weight: 50
---

### Talking to real-time device drivers

Using the [EVL kernel API]({{< relref "core/kernel-api/_index.md"
>}}), you can extend an existing character driver for supporting
out-of-band I/O, or even write one from scratch.

On the user side, application can exchange data with, send requests to
these real-time capable drivers from the out-of-band execution stage
with the a couple of additional services [libevl]({{< relref
"core/user-api/_index.md" >}}) provides.

{{% notice note %}}
The EVL core does not currently support out-of-band socket
semantics. Maybe at some point this will happen, but this needs more
thought about proper integration of this feature. Not there yet.
{{% /notice %}}

You may notice that several POSIX file I/O services such as
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html),
[close(2)](http://man7.org/linux/man-pages/man2/close.2.html),
[fcntl(2)](http://man7.org/linux/man-pages/man2/fcntl.2.html),
[mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html) and so on
have no out-of-band counterpart in the following list. The reason is
that we don't need them: opening, closing or mapping a file are
inherently non-deterministic operations, which may block for an
unspecified amount of time for a number of reasons, depending on the
underlying file and current runtime conditions. Besides, those are
hardly useful in a time-critical loop.

However, issuing data transfers and control requests to the driver is
definitely something we may want to happen within a bounded time,
hence directly from the out-of-band execution stage.

{{% notice tip %}}
Since the EVL core exports each [element]({{< relref
"core/_index.md#evl-core-elements" >}}) as a character device which
can be accessed from _/dev/evl_, interacting with EVL elements via
[libevl]({{< relref "core/user-api/_index.md" >}}) is a actually done
through the out-of-band I/O interface documented here.
{{% /notice %}}

### Out-of-band I/O services

{{< proto oob_read >}}
ssize_t oob_read(int efd, void *buf, size_t count)
{{< /proto >}}

This is the strict equivalent to the standard
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html) system
call, but for running the request from the out-of-band stage. In other
words, `oob_read()` attempts to read up to _count_ bytes from file
descriptor _fd_ into the buffer starting at _buf_, from the
out-of-band execution stage.

The caller must be an [EVL thread]({{< relref
"core/user-api/thread/_index.md" >}}), which may be switched
automatically by the EVL core to the out-of-band execution stage as a
result of this call.

{{% argument fd %}}
A file descriptor obtained by opening a [real-time
capable driver]({{< relref "core/kernel-api/_index.md" >}}) which we
want to read from.
{{% /argument %}}

{{% argument buf %}}
A buffer to receive the data.
{{% /argument %}}

{{% argument count %}}
The number of bytes to read at most, which should fit into _buf_.
{{% /argument %}}

`oob_read()` returns the actual number of bytes read, copied to _buf_
on success. Otherwise, -1 is returned, and errno is set to the error
code:

EBADF	if _fd_ does not refer to a real-time capable driver, or _fd_
	was not opened for reading.

EINVAL  if _fd_ does not support the [.oob_read operation]({{< relref
	"core/kernel-api/_index.md" >}}).

EFAULT	if _buf_ points to invalid memory.

EAGAIN	_fd_ is marked as non-blocking (O_NONBLOCK), and the read would
	block.

Other driver-specific error codes may be returned, such as:

ENOBUFS _fd_ is a [cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md" >}}) file descriptor, and there
	is no ring buffer space associated with the outbound traffic
	(i.e. _o\_bufsz_ parameter was zero when [creating the
	cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md#evl_new_xbuf" >}})).

EINVAL  _fd_ is a [cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md" >}}) file descriptor, and
	_count_ is greater than the size of the ring buffer associated
	with the traffic direction. (i.e. either the _i\_bufsz_ or
	_o\_bufsz_ parameter given when [creating the
	cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md#evl_new_xbuf" >}})).

---

{{< proto oob_write >}}
ssize_t oob_write(int efd, const void *buf, size_t count)
{{< /proto >}}

This is the strict equivalent to the standard
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) system
call, but for running the request from the out-of-band stage. In other
words, `oob_write()` attempts to write up to _count_ bytes to file
descriptor _fd_ from the buffer starting at _buf_, from the
out-of-band execution stage.

The caller must be an [EVL thread]({{< relref
"core/user-api/thread/_index.md" >}}), which may be switched
automatically by the EVL core to the out-of-band execution stage as a
result of this call.

{{% argument fd %}}
A file descriptor obtained by opening a [real-time
capable driver]({{< relref "core/kernel-api/_index.md" >}}) which we
want to read from.
{{% /argument %}}

{{% argument buf %}}
A buffer containing the data to be written.
{{% /argument %}}

{{% argument count %}}
The number of bytes to write starting from _buf_.
{{% /argument %}}

`oob_write()` returns the actual number of bytes written from _buf_ on
success. Otherwise, -1 is returned, and errno is set to the error
code:

EBADF	if _fd_ does not refer to a real-time capable driver, or _fd_ was
	not opened for writing.

EINVAL  if _fd_ does not support the [.oob_write operation]({{< relref
	"core/kernel-api/_index.md" >}}).

EFAULT	if _buf_ points to invalid memory.

EAGAIN	_fd_ is marked as non-blocking (O_NONBLOCK), and the write would
	block.

Other driver-specific error codes may be returned, such as:

EFBIG	_fd_ is a [proxy]({{< relref "core/user-api/proxy/_index.md"
	>}}) file descriptor, and _count_ is larger than the size of the
	output buffer as specified in the call to [evl_new_proxy()]
	({{< relref "core/user-api/proxy/_index.md#evl_new_proxy" >}}).

EINVAL	_fd_ is a [proxy]({{< relref "core/user-api/proxy/_index.md"
	>}}) file descriptor, and _count_ is not a multiple of the
	output granularity as specified in the call to [evl_new_proxy()]
	({{< relref "core/user-api/proxy/_index.md#evl_new_proxy" >}}).

ENOBUFS _fd_ is a [cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md" >}}) file descriptor, and there is no
	ring buffer space associated with the inbound traffic (i.e. _i\_bufsz_
	parameter was zero when [creating the cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md#evl_new_xbuf" >}})).

EINVAL  _fd_ is a [cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md" >}}) file descriptor, and
	_count_ is greater than the size of the ring buffer associated
	with the traffic direction. (i.e. either the _i\_bufsz_ or
	_o\_bufsz_ parameter given when [creating the
	cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md#evl_new_xbuf" >}})).

---

{{< proto oob_ioctl >}}
int oob_ioctl(int efd, unsigned long request, ...)
{{< /proto >}}

This is the strict equivalent to the standard
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) system
call, but for running the I/O control request from the out-of-band
stage. In other words, `oob_ioctl()` issues _request_ to file
descriptor _fd_ from the out-of-band execution stage.

The caller must be an [EVL thread]({{< relref
"core/user-api/thread/_index.md" >}}), which may be switched
automatically by the EVL core to the out-of-band execution stage as a
result of this call.

{{% argument fd %}}
A file descriptor obtained by opening a [real-time
capable driver]({{< relref "core/kernel-api/_index.md" >}}) which we
want to send a request to.
{{% /argument %}}

{{% argument request %}}
The I/O control request code.
{{% /argument %}}

{{% argument "..." %}}
An optional variable argument list which applies to _request_.
{{% /argument %}}

`oob_ioctl()` returns zero on success. Otherwise, -1 is returned, and
errno is set to the error code:

EBADF	if _fd_ does not refer to a real-time capable driver.

ENOTTY  if _fd_ does not support the [.oob_ioctl operation]({{< relref
	"core/kernel-api/_index.md" >}}), or the driver does not
	implement _request_.

EFAULT	if _buf_ points to invalid memory.

EAGAIN	_fd_ is marked as non-blocking (O_NONBLOCK), and the read would
	block.

Other driver-specific error codes may be returned.
