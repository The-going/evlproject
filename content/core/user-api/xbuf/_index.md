---
menuTitle: "Cross-buffer"
weight: 35
---

### Bi-directional out-of-band &#8660; in-band messaging

A most typical requirement in dual kernel systems is to exchange data
between threads running in separate execution stages, i.e. out-of-band
vs in-band, without incuring any delay for the out-of-band side.  The
EVL core implements a feature called a _cross-buffer_ (aka _xbuf_),
which connects the sending out-of-band end of a communication channel
to its receiving in-band end, and conversely. As a result, data sent
to a cross-buffer by calling [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) are received by calling
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html), data sent
by calling
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) are
received by calling [oob_read()]({{< relref
"core/user-api/io/_index.md#oob_read" >}}). A cross-buffer can be used
for transferring fixed-size or variable-size data, supports
message-based and byte-oriented streams. For instance, such a
mechanism would enable a GUI front-end to receive monitoring data from
a real-time work loop.

![Alt text](/images/xbuf.png "Cross-buffer deadlock prevention")

#### xbuf-specific semantics for sending and receiving

The following rules apply to cross-buffer transfers, in either
directions (inbound and outbound):

- If [O_NONBLOCK](http://man7.org/linux/man-pages/man2/fcntl.2.html)
  is clear on the cross-buffer file descriptor, there
  are no short reads on the receiving end: a reader always gets a
  complete message of the requested length, blocking if necessary
  _except_ if a sender is currently blocked on the other end of the
  channel, waiting for some space to be available into the ring buffer
  to complete a write operation. In this case, a short read is
  performed to prevent a deadlock, which means the reader may receive
  less data than requested in the read call. This situation arises
  when the size picked for the ring buffer does not fit the transfer
  pattern, such as follows:

![Alt text](/images/xbuf-deadlock.png "Cross-buffer deadlock prevention")

{{% notice tip %}}
If you plan to send fixed-size messages through a cross-buffer and
keeping the message boundaries intact matters, you should pick a
buffer size which is a multiple of the basic message size, reading
and writing complete messages at each transfer.
{{% /notice %}}

  Otherwise, if [O_NONBLOCK](http://man7.org/linux/man-pages/man2/fcntl.2.html)
  is set and not enough bytes are immediately
  available from the ring buffer for satisfying the request, the write
  operation fails with the `EAGAIN` error status.

- There are no short or scattered writes to the ring buffer: the
  operation either succeeds immediately or blocks until enough space
  is available into the buffer for copying the entire message
  atomically, unless [O_NONBLOCK](http://man7.org/linux/man-pages/man2/fcntl.2.html)
  is set on the cross-buffer file descriptor (`EAGAIN`).

### Cross-buffer services {#xbuf-services}

{{< proto evl_create_xbuf >}}
int evl_create_xbuf(size_t i_bufsz, size_t o_bufsz, int flags, const char *fmt, ...)
{{< /proto >}}

This call creates a new cross-buffer, then returns a file descriptor
representing the communication channel upon success. Internally, a
cross-buffer is implemented as a pair of ring buffers conveying data
from in-band to out-of-band context (i.e. _inbound_ traffic), and
conversely (i.e. _outbound_ traffic). [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) and [oob_read()]({{<
relref "core/user-api/io/_index.md#oob_read" >}}) should be used by
callers running on the out-of-band stage for sending/receiving data
through this channel respectively. Conversely,
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) and
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html) should be
used for accessing the channel from the in-band stage.

{{% argument i_bufsz %}}
The size in bytes of the ring buffer conveying inbound traffic. If
zero, the cross-buffer is intended to relay outbound messages
exclusively, i.e. from the in-band to the out-of-band context.
_i\_bufsz_ must not exceed 2^30.
{{% /argument %}}

{{% argument o_bufsz %}}
The size in bytes of the ring buffer conveying outbound traffic. If
zero, the cross-buffer is intended to relay inbound messages
exclusively, i.e. from the out-of-band to the in-band context.
_i\_bufsz_ must not exceed 2^30.
{{% /argument %}}

{{% argument flags %}}
A set of creation flags for the new element, defining its
[visibility]({{< relref "core/user-api/_index.md#element-visibility"
>}}):

  - `EVL_CLONE_PUBLIC` denotes a public element which is represented
    by a device file in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy, which
    makes it visible to other processes for sharing.
  
  - `EVL_CLONE_PRIVATE` denotes an element which is private to the
    calling process. No device file appears for it in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy.

  - `EVL_CLONE_NONBLOCK` sets the file descriptor of the new cross-buffer in
    non-blocking I/O mode (`O_NONBLOCK`). By default, `O_NONBLOCK` is
    cleared for the file descriptor.
 {{% /argument %}}

{{% argument fmt %}}
A [printf](http://man7.org/linux/man-pages/man3/printf.3.html)-like
format string to generate the cross-buffer name. See this description of the
[naming convention]
({{< relref "core/user-api/_index.md#element-naming-convention" >}}).
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_create_xbuf()]({{< relref "#evl_create_xbuf" >}}) returns the
file descriptor of the new cross-buffer on success. If the call fails,
a negated error code is returned instead:

- -EEXIST	The generated name is conflicting with an existing cross-buffer name.

- -EINVAL	Either _flags_ is wrong,  _i\_bufsz_ and/or _o\_bufsz_ are wrong,
  		or the [generated name]({{< relref "core/user-api/_index.md#element-naming-convention"
  		>}}) is badly formed.

- -ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

- -ENXIO	The EVL library is not initialized for the current process.
  		Such initialization happens implicitly when
  		[evl_attach_self()]({{% relref
  		"core/user-api/thread/_index.md#evl_attach_self" %}})
  		is called by any thread of your process, or by
  		explicitly calling [evl_init()]({{< relref
  		"core/user-api/init/_index.md#evl_init" >}}). You have
  		to bootstrap the library services in a way or another
  		before creating a cross-buffer.

---

{{< proto evl_new_xbuf >}}
int evl_new_xbuf(size_t io_bufsz, const char *fmt, ...)
{{< /proto >}}

This call is a shorthand for creating a private cross-buffer with
identically-sized I/O buffers. It is identical to calling:

```
	evl_create_xbuf(io_bufsz, io_bufsz, EVL_CLONE_PRIVATE, fmt, ...);
```

{{% notice info %}}
Note that if the [generated name] ({{< relref
"core/user-api/_index.md#element-naming-convention" >}}) starts with a
slash ('/') character, `EVL_CLONE_PRIVATE` would be automatically turned
into `EVL_CLONE_PUBLIC` internally.
{{% /notice %}}

---

### Events pollable from a cross-buffer descriptor {#xbuf-poll-events}

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}})
interface can monitor the following events occurring on a cross-buffer
file descriptor:

- _POLLIN_ and _POLLRDNORM_ are set whenever data coming from the
  in-band side is available for reading by a call to [oob_read()]({{<
  relref "core/user-api/io/_index.md#oob_read" >}}).

- _POLLOUT_ and _POLLWRNORM_ are set whenever there is still room in
  the output ring buffer for sending more data to the in-band side
  using [oob_write()]({{< relref
  "core/user-api/io/_index.md#oob_write" >}}).

Conversely, you can also use the in-band
[poll(2)](http://man7.org/linux/man-pages/man2/poll.2.html) on a
cross-buffer file descriptor, monitoring the following events:

- _POLLIN_ and _POLLRDNORM_ are set whenever data coming from the
  out-of-band side is available for reading, by a call to
  [read(2)](http://man7.org/linux/man-pages/man2/read.2.html).
  
- _POLLOUT_ and _POLLWRNORM_ are set whenever there is still room in
  the output ring buffer for sending more data to the out-of-band
  side, using
  [write(2)](http://man7.org/linux/man-pages/man2/write.2.html).

---

{{<lastmodified>}}
