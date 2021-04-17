---
title: "Out-of-band I/O services"
weight: 50
---

### Talking to real-time capable device drivers

Using the [EVL kernel API]({{< relref "core/kernel-api/_index.md"
>}}), you can extend an existing driver for supporting out-of-band I/O
operations, or even write one from scratch. Both character-based I/O
and socket protocol drivers are supported.

On the user side, application can exchange data with, send requests to
these real-time capable drivers from the out-of-band execution stage
with the a couple of additional services [libevl]({{< relref
"core/user-api/_index.md" >}}) provides.

You may notice that several POSIX file I/O services such as
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html),
[socket(2)](http://man7.org/linux/man-pages/man2/socket.2.html),
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
Since the EVL core exports every [public element]({{< relref "core/user-api/_index.md#element-visibility" >}})
as a character device which can be accessed from
[/dev/evl]({{< relref "core/user-api/_index.md#evl-fs-hierarchy" >}}),
[libevl]({{< relref "core/user-api/_index.md" >}}) can interface with
elements from other processes through the out-of-band I/O requests
documented here, which are sent to the corresponding devices.
{{% /notice %}}

### Out-of-band I/O services {#oob-io-services}

{{< proto oob_read >}}
ssize_t oob_read(int efd, void *buf, size_t count)
{{< /proto >}}

This is the strict equivalent to the standard
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html) system
call, for sending the request from the out-of-band stage to an EVL
driver. In other words, [oob_read()]({{< relref "#oob_read" >}})
attempts to read up to _count_ bytes from file descriptor _fd_ into
the buffer starting at _buf_, from the out-of-band execution stage.

The caller must be an [EVL thread]({{< relref
"core/user-api/thread/_index.md" >}}), which may be switched
automatically by the EVL core to the out-of-band execution stage as a
result of this call.

{{% argument efd %}}
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

[oob_read()]({{< relref "#oob_read" >}}) returns the actual number of
bytes read, copied to _buf_ on success. Otherwise, -1 is returned, and
errno is set to the error code:

EBADF	if _fd_ does not refer to a real-time capable driver, or _fd_
	was not opened for reading.

EINVAL  if _fd_ does not support the [.oob_read operation]({{< relref
	"core/kernel-api/_index.md" >}}).

EFAULT	if _buf_ points to invalid memory.

EAGAIN	_fd_ is marked as [non-blocking (O_NONBLOCK)]
	(http://man7.org/linux/man-pages/man2/fcntl.2.html), and the read
	operation would block.

Other driver-specific error codes may be returned, such as:

ENOBUFS _fd_ is a [cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md" >}}) file descriptor, and there
	is no ring buffer space associated with the outbound traffic
	(i.e. _o\_bufsz_ parameter was zero when [creating the
	cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md#evl_create_xbuf" >}})).

EINVAL  _fd_ is a [cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md" >}}) file descriptor, and
	_count_ is greater than the size of the ring buffer associated
	with the traffic direction. (i.e. either the _i\_bufsz_ or
	_o\_bufsz_ parameter given when [creating the
	cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md#evl_create_xbuf" >}})).

ENXIO	_fd_ is a [proxy]({{< relref "core/user-api/proxy/_index.md"
	>}}) file descriptor which is not available for input. See
	[EVL_CLONE_INPUT]({{< relref
	"core/user-api/proxy/_index.md#evl_create_proxy" >}}).

---

{{< proto oob_write >}}
ssize_t oob_write(int efd, const void *buf, size_t count)
{{< /proto >}}

This is the strict equivalent to the standard
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) system
call, for sending the request from the out-of-band stage to an EVL
driver. In other words, [oob_write()]({{< relref "#oob_write" >}})
attempts to write up to _count_ bytes to file descriptor _fd_ from the
buffer starting at _buf_, from the out-of-band execution stage.

The caller must be an [EVL thread]({{< relref
"core/user-api/thread/_index.md" >}}), which may be switched
automatically by the EVL core to the out-of-band execution stage as a
result of this call.

{{% argument efd %}}
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

[oob_write()]({{< relref "#oob_write" >}}) returns the actual number
of bytes written from _buf_ on success. Otherwise, -1 is returned, and
errno is set to the error code:

EBADF	if _fd_ does not refer to a real-time capable driver, or _fd_ was
	not opened for writing.

EINVAL  if _fd_ does not support the [.oob_write operation]({{< relref
	"core/kernel-api/_index.md" >}}).

EFAULT	if _buf_ points to invalid memory.

EAGAIN	_fd_ is marked as [non-blocking (O_NONBLOCK)]
	(http://man7.org/linux/man-pages/man2/fcntl.2.html), and the
	write operation would block.

Other driver-specific error codes may be returned, such as:

EFBIG	_fd_ is a [proxy]({{< relref "core/user-api/proxy/_index.md"
	>}}) file descriptor, and _count_ is larger than the size of the
	output buffer as specified in the call to [evl_create_proxy()]
	({{< relref "core/user-api/proxy/_index.md#evl_create_proxy" >}}).

EINVAL	_fd_ is a [proxy]({{< relref "core/user-api/proxy/_index.md"
	>}}) file descriptor, and _count_ is not a multiple of the
	output granularity as specified in the call to [evl_create_proxy()]
	({{< relref "core/user-api/proxy/_index.md#evl_create_proxy" >}}).

ENOBUFS _fd_ is a [cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md" >}}) file descriptor, and there is no
	ring buffer space associated with the inbound traffic (i.e. _i\_bufsz_
	parameter was zero when [creating the cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md#evl_create_xbuf" >}})).

EINVAL  _fd_ is a [cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md" >}}) file descriptor, and
	_count_ is greater than the size of the ring buffer associated
	with the traffic direction. (i.e. either the _i\_bufsz_ or
	_o\_bufsz_ parameter given when [creating the
	cross-buffer]({{< relref
	"core/user-api/xbuf/_index.md#evl_create_xbuf" >}})).

ENXIO	_fd_ is a [proxy]({{< relref "core/user-api/proxy/_index.md"
	>}}) file descriptor which is not available for output. See
	[EVL_CLONE_OUTPUT]({{< relref
	"core/user-api/proxy/_index.md#evl_create_proxy" >}}).

---

{{< proto oob_ioctl >}}
int oob_ioctl(int efd, unsigned long request, ...)
{{< /proto >}}

This is the strict equivalent to the standard
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) system
call, for sending the I/O control request from the out-of-band stage
to an EVL driver. In other words, [oob_ioctl()]({{< relref
"#oob_ioctl" >}}) issues _request_ to file descriptor _fd_ from the
out-of-band execution stage.

The caller must be an [EVL thread]({{< relref
"core/user-api/thread/_index.md" >}}), which may be switched
automatically by the EVL core to the out-of-band execution stage as a
result of this call.

{{% argument efd %}}
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

[oob_ioctl()]({{< relref "#oob_ioctl" >}}) returns zero on
success. Otherwise, -1 is returned, and errno is set to the error
code:

EBADF	if _fd_ does not refer to a real-time capable driver.

ENOTTY  if _fd_ does not support the [.oob_ioctl operation]({{< relref
	"core/kernel-api/_index.md" >}}), or the driver does not
	implement _request_.

EFAULT	if _buf_ points to invalid memory.

EAGAIN  _fd_ is marked as [non-blocking (O_NONBLOCK)](http://man7.org/linux/man-pages/man2/fcntl.2.html), and the control
	request would block.

Other driver-specific error codes may be returned.

---

{{< proto oob_recvmsg >}}
ssize_t oob_recvmsg(int s, struct oob_msghdr *msghdr, const struct timespec *timeout, int flags)
{{< /proto >}}

This is an equivalent to the standard
[recvmsg(2)](http://man7.org/linux/man-pages/man2/recvmsg.2.html)
system call, for sending the request from the out-of-band stage to an
EVL driver with a socket-based interface.  In other words,
[oob_recvmsg()]({{< relref "#oob_recvmsg" >}}) is used to receive
messages from an out-of-band capable EVL socket from the out-of-band
execution stage.

The caller must be an [EVL thread]({{< relref
"core/user-api/thread/_index.md" >}}), which may be switched
automatically by the EVL core to the out-of-band execution stage as a
result of this call.

{{% argument s %}}
A socket descriptor obtained from a regular
[socket(2)](http://man7.org/linux/man-pages/man2/recvmsg.2.html) call,
with the `SOCK_OOB` flag set in the _type_ argument, denoting that
out-of-band services are enabled for the socket.
{{% /argument %}}

{{% argument msghdr %}}
A pointer to a structure containing the multiple arguments to this
call, [which is described below]({{< relref "#oob-msghdr" >}}).
{{% /argument %}}

{{% argument timeout %}}
A time limit to wait for a message before the call returns on
error. The built-in clock [EVL_CLOCK_MONOTONIC]({{< relref
"core/user-api/clock/_index.md#builtin-clocks" >}}) is used for
tracking the elapsed time. If NULL is passed, the call is allowed
to wait indefinitely for a message.
{{% /argument %}}

{{% argument flags %}}
A set of flags further qualifying the operation. Only the following
flags should be recognized for out-of-band requests:

- `MSG_DONTWAIT` causes the call to fail with the error EAGAIN if no
message is immediately available at the time of the
call. `MSG_DONTWAIT` is implied if O_NONBLOCK was set for the socket
descriptor via the
[fcntl(2)](http://man7.org/linux/man-pages/man2/fcntl.2.html) F_SETFL
operation.

- `MSG_PEEK` causes the receive operation to return data from the
beginning of the receive queue without removing that data from the
queue.  Thus, a subsequent receive call will return the same data.
{{% /argument %}}

[oob_recvmsg()]({{< relref "#oob_recvmsg" >}}) returns the actual
number of bytes received on success. Otherwise, -1 is returned, and
errno is set to the error code:

EBADF   if _s_ does not refer to a valid socket opened with the
	`SOCK_OOB` type flag set, or _s_ was not opened for reading.

EINVAL  if _s_ does not support the [.oob_ioctl operation]({{< relref
	"core/kernel-api/_index.md" >}}).

EFAULT	if _msghdr_, or any buffer it refers to indirectly points to
	invalid memory.

EAGAIN	_s_ is marked as [non-blocking (O_NONBLOCK)](http://man7.org/linux/man-pages/man2/fcntl.2.html), or
	`MSG_DONTWAIT` is set in _flags_, and the receive operation would
	block.

ETIMEDOUT  the _timeout_ fired before the operation could complete successfully.

---

{{< proto oob_sendmsg >}}
ssize_t oob_sendmsg(int s, const struct oob_msghdr *msghdr, const struct timespec *timeout, int flags)
{{< /proto >}}

This call is equivalent to the standard
[sendmsg(2)](http://man7.org/linux/man-pages/man2/sendmsg.2.html)
system call, for sending the request from the out-of-band stage to an
EVL driver with a socket-based interface.  In other words,
[oob_sendmsg()]({{< relref "#oob_sendmsg" >}}) is used to send
messages to an out-of-band capable EVL socket from the out-of-band
execution stage.

The caller must be an [EVL thread]({{< relref
"core/user-api/thread/_index.md" >}}), which may be switched
automatically by the EVL core to the out-of-band execution stage as a
result of this call.

{{% argument s %}}
A socket descriptor obtained from a regular
[socket(2)](http://man7.org/linux/man-pages/man2/recvmsg.2.html) call,
with the `SOCK_OOB` flag set in the _type_ argument, denoting that
out-of-band services are enabled for the socket.
{{% /argument %}}

{{% argument msghdr %}}
A pointer to a structure containing the multiple arguments to this
call, [which is described below]({{< relref "#oob-msghdr" >}}).
{{% /argument %}}

{{% argument timeout %}}
A time limit to wait for an internal buffer to be available for
sending the message before the call returns on error. The built-in
clock [EVL_CLOCK_MONOTONIC]({{< relref
"core/user-api/clock/_index.md#builtin-clocks" >}}) is used for
tracking the elapsed time.  If NULL is passed, the call is allowed
 to wait indefinitely for a buffer.
{{% /argument %}}

{{% argument flags %}}
A set of flags further qualifying the operation. Only the following
flag should be recognized for out-of-band requests:

- `MSG_DONTWAIT` causes the call to fail with the error EAGAIN if no
buffer is immediately available at the time of the call for sending
the message. `MSG_DONTWAIT` is implied if O_NONBLOCK was set for the
socket descriptor via the
[fcntl(2)](http://man7.org/linux/man-pages/man2/fcntl.2.html) F_SETFL
operation.
{{% /argument %}}

[oob_sendmsg()]({{< relref "#oob_sendmsg" >}}) returns the actual
number of bytes sent on success. Otherwise, -1 is returned, and errno
is set to the error code:

EBADF	if _s_ does not refer to a valid socket opened with the
	`SOCK_OOB` type flag set, or _s_ was not opened for writing.

EINVAL  if _s_ does not support the [.oob_ioctl operation]({{< relref
	"core/kernel-api/_index.md" >}}).

EFAULT	if _msghdr_, or any buffer it refers to indirectly points to
	invalid memory.

EAGAIN	_s_ is marked as [non-blocking (O_NONBLOCK)](http://man7.org/linux/man-pages/man2/fcntl.2.html), or
	`MSG_DONTWAIT` is set in _flags_, and the send operation would
	block.

ETIMEDOUT  the _timeout_ fired before the operation could complete successfully.

--

## The out-of-band message header {#oob-msghdr}

The structure `oob_msghdr` which is passed to the [oob_recvmsg()]({{<
relref "#oob_recvmsg" >}}) and [oob_sendmsg()]({{< relref
"#oob_sendmsg" >}}) calls is defined as follows:

           struct oob_msghdr {
               void           *msg_name;       /* Optional address */
               socklen_t       msg_namelen;    /* Size of address */
               struct iovec   *msg_iov;        /* Scatter/gather array */
               size_t          msg_iovlen;     /* # elements in msg_iov */
               void           *msg_control;    /* Ancillary data, see below */
               size_t          msg_controllen; /* Ancillary data buffer len */
               int             msg_flags;      /* Flags on received message */
               struct timespec msg_time;       /* Optional time, see below */
           };

           struct iovec {                    /* Scatter/gather array items */
               void  *iov_base;              /* Starting address */
               size_t iov_len;               /* Number of bytes to transfer */
           };

The `msg_name` field points to a caller-allocated buffer that is used
to return the source address if the socket is unconnected.  The caller
should set `msg_namelen` to the size of this buffer before this call.
On success, [oob_recvmsg()]({{< relref "#oob_recvmsg" >}}) updates
`msg_namelen` to contain the length of the returned address.  If the
application does not need to know the source address, `msg_name` can
be specified as NULL.

The fields `msg_iov` and `msg_iovlen` describe scatter-gather
locations pointing at the message data being sent or received, as
discussed in
[readv(2)](http://man7.org/linux/man-pages/man2/readv.2.html).

The field `msg_control` points to a buffer for other protocol
control-related messages or miscellaneous ancillary data.  When either
[oob_recvmsg()]({{< relref "#oob_recvmsg" >}}) or [oob_sendmsg()]({{<
relref "#oob_sendmsg" >}}) is called, `msg_controllen` should contain
the length of the available buffer in `msg_control`. On success,
[oob_recvmsg()]({{< relref "#oob_recvmsg" >}}) updates
`msg_controllen` to contain the actual length of the control message
sequence returned by the call.

The `msg_flags` field is only set on return of [oob_recvmsg()]({{<
relref "#oob_recvmsg" >}}).  It can contain any of the flags which may
be returned by
[recvmsg(2)](http://man7.org/linux/man-pages/man2/recvmsg.2.html).

`msg_time` may be used to send or receive timestamping information
to/from the protocol driver implementing out-of-band operations.

{{% notice warning %}}
Protocol drivers should no attach any meaning to `MSG_OOB` when
operating in out-of-band mode, so that no additional confusion arises
with the common usage of this flag with
[recvmsg(2)](http://man7.org/linux/man-pages/man2/recvmsg.2.html).
{{% /notice %}}

---

{{<lastmodified>}}
