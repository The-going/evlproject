---
title: "File proxy"
weight: 40
---

### Zero-latency output to regular files

A common issue with dual kernel systems stems from the requirement NOT
to issue [inband system calls]({{< relref
"dovetail/altsched.md#inband-switch" >}}) while running time-critical
code in out-of-band context. Problem is that sometimes, you may need
to write to regular files such as logging information to the console
or elsewhere from an out-of-band work loop. Doing so by calling common
[stdio(3)](http://man7.org/linux/man-pages/man3/stdio.3.html) routines
directly is therefore not an option for latency reasons.

The EVL core solves this problem with the _file proxy_ feature, which
can channel the output sent through a proxy file descriptor to some
arbitrary destination file descriptor associated with the former,
keeping the caller on the out-of-band execution stage. This works by
buffering the output in a circular buffer, offloading the actual write
to an internal worker thread running inband which relays the content
of this ring buffer to the target file represented by the destination
descriptor. For this reason, such output can be delayed until the
inband stage resumes on the current CPU.  For instance,
[evl_printf()]({{< relref "#evl_printf" >}}) formats then emits the
resulting string to _fileno(stdout)_ via an internal proxy created by
`libevl.so` at initialization.

Typically, out-of-band writers send data through the proxy file
descriptor using EVL's [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) system call, which inband
readers can receive using the
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html) system
call. You can associate any type of file with a proxy, including a
[socket(2)](http://man7.org/linux/man-pages/man2/socket.2.html),
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html),
[signalfd(2)](http://man7.org/linux/man-pages/man2/signalfd.2.html),
[pipe(2)](http://man7.org/linux/man-pages/man7/pipe.7.html) and so on
(see also the discussion below about the write [granularity]({{<
relref "#evl_new_proxy" >}})). This means that you could also use a
proxy for signaling an inband object from the out-of-band context, if
doing so can be done using a regular
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) system
call, like
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html) and
[signalfd(2)](http://man7.org/linux/man-pages/man2/signalfd.2.html).

The proxy also handles output from callers running on the inband stage
transparently. In this case, a regular
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) can be
used to channel the data to the target file.

![Alt text](/images/proxy.png "File proxy")

> Logging debug messages via a proxy

For instance, you may want your application to dump debug information
to some arbitray file as it runs, including when running time-critical
code out-of-band. Admittedly, this would add some overhead, but still
low enough to keep the system happy, while giving you precious debug
hints. Obviously, you cannot get away with calling the plain
[printf(3)](http://man7.org/linux/man-pages/man3/printf.3.html)
service for this, since it would downgrade the calling EVL thread to
[inband mode]({{< relref "dovetail/altsched.md#inband-switch"
>}}). However, you may create a proxy to the debug file:

```
	#include <fcntl.h>
	#include <evl/proxy.h>

	int proxyfd, debugfd;

	debugfd = open("/tmp/debug.log", O_WRONLY|O_CREAT|O_TRUNC, 0600);
	...
	/* Create a proxy offering a 1 MB buffer. */
	proxyfd = evl_new_proxy(debugfd, 1024*1024, 0, "log:%d", getpid());
	...
```

Then channel debug output from anywhere in your code you see fit
through the proxy this way:

```
	#include <evl/proxy.h>

	evl_print_proxy(proxyfd, "some useful debug information");
```

> Synchronizing inband threads on out-of-band events with via a proxy

There are a couple of ways which you could use in order to wake up an
inband thread waiting for some event to occur on the out-of-band side
of your application, while preventing the waker from being demoted as
a result of sending such a signal. One would involve running the
inband thread in the [SCHED_WEAK scheduling class]({{< relref
"core/user-api/scheduling/_index.md#SCHED_WEAK" >}}), waiting on some
EVL synchronization object, such as a [semaphore]({{< relref
"core/user-api/semaphore/_index.md" >}}) or [event flag group]({{<
relref "core/user-api/event/_index.md" >}}). Another one would use a
[cross-buffer]({{< relref "core/user-api/xbuf/_index.md" >}}) for
sending some wakeup datagram from the out-of-band caller to the inband
waiter.

The file proxy can also help in implementing such construct, by
connecting it to a regular
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html),
which can be signaled using the common
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) system
call:

```
	#include <sys/types.h>
	#include <sys/eventfd.h>
	#include <evl/proxy.h>

	int evntfd;

	evntfd = eventfd(0, EFD_SEMAPHORE);
	...
	/* Create the proxy, allow up to 3 notifications to pile up. */
	proxyfd = evl_new_proxy(evntfd, sizeof(uint64_t) * 3, sizeof(uint64_t), "event-relay");
	...
```

Note the **specific granularity value** mentioned in the creation call
above: we do want the proxy to write 64-bit values at each transfer,
in order to cope with the requirements of the
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html)
interface. Once the proxy is created, set up for routing all write
requests to the regular event file descriptor by 64-bit chunks, the
inband thread can wait for out-of-band events reading from the other
side of the channel as follows:

```
	#include <stdint.h>
	#include <evl/proxy.h>

	void oob_waker(int proxyfd)
	{
		uint64 val = 1;
		ssize_t ret;
	
		ret = oob_write(proxyfd, &val, sizeof(val));
		...
	}

	void inband_sleeper(int evntfd)
	{
		uint64 val = 1;
		ssize_t ret;
		
		for (;;) {
		    /* Wait for the next wakeup signal. */
		    ret = read(evntfd, &val, sizeof(val));
		    ...
		}
	}
```

### Export of memory mappings

EVL proxies can be also be used for carrying over memory mapping
requests to a final device which actually serves the mapping.  The
issuer of such request does not have to know which device driver will
be actually handling that request: the proxy acts as an anchor point
agreed between peer processes in order to locate a particular
memory-mappable resource, and the proxy simply redirects the mapping
requests it receives to the device driver handling the target file it
is proxying for.

![Alt text](/images/proxy-mmap.png "Mapping proxy")

> Exporting process-private memfd memory to the outer world

For instance, this usage of a proxy comes in handy when you need to
export a private memory mapping like those obtained by
[memfd_create(2)](http://man7.org/linux/man-pages/man2/memfd_create.2.html)
to a peer process, assuming you don't want to deal with the hassle of
the POSIX
[shm_open(3)](http://man7.org/linux/man-pages/man3/shm_open.3.html)
interface for sharing memory objects.  In this case, all this peer has
to know is the name of the proxy which is associated with the
memory-mappable device. It can
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html) that proxy
device in _/dev/evl/proxy/_, then issue the
[mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html) system
call for receiving a mapping to the backing device's memory. From the
application standpoint, creating then sharing a 1 KB RAM segment with
other peer processes may be as simple as this:

```
	#include <sys/types.h>
	#include <sys/memfd.h>
	#include <sys/mman.h>
	#include <unistd.h>
	#include <evl/proxy.h>

	int memfd, ret;
	void *ptr;

	memfd = memfd_create("whatever", 0);
	...
	/* Set the size of the underlying segment then map it locally. */
	ret = ftruncate(memfd, 1024);
	ptr = mmap(NULL, 1024, PROT_READ|PROT_WRITE, MAP_SHARED, memfd, 0);
	...
	/* Create a proxy others can find in the /dev/evl/proxy hierarchy. */
	proxyfd = evl_new_proxy(memfd, 0, 0, "some-fancy-proxy");
	...
```

Any remote process peer could then do:

```
	#include <sys/types.h>
	#include <sys/mman.h>
	#include <fcntl.h>

	void *ptr;
	int fd;

	/* Open the proxy device relaying mapping requests to memfd(). */
	fd = open("/dev/evl/proxy/some-fancy-proxy", O_RDWR);
	...
	/* Map the private RAM segment created by our peer. */
	ptr = mmap(NULL, 1024, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
	...
```

### Proxy services {#proxy-services}

{{< proto evl_new_proxy >}}
int evl_new_proxy(int targetfd, size_t bufsz, size_t granularity, const char *fmt, ...)
{{< /proto >}}

This call creates a proxy, returning a file descriptor representing
the new object upon success. [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) should be used to send
data through the proxy for zero-latency output to regular files.

{{% argument targetfd %}}
A descriptor referring to the destination file which should receive
the output written to the proxy file descriptor returned by
`evl_new_proxy()`.
{{% /argument %}}

{{% argument bufsz %}}
The size in bytes of the ring buffer where the output data is kept
until the inband worker has relayed it to the destination file.
Out-of-band writers may block attempting to write to a proxy file
descriptor if the output buffer is full, waiting to be depleted by the inband
worker thread, unless the proxy descriptor is set to non-blocking mode
(O_NONBLOCK). Zero is an acceptable value if you plan to use this
proxy exclusively for exporting a shared memory mapping, in which case
there is no point in reserving an output buffer.
{{% /argument %}}

{{% argument granularity %}}
In some cases, the target file may have special semantics, which
requires a fixed amount of data to be submitted at each write
operation, like the
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html) file
which requires 64-bit words to be written to it at each transfer. When
granularity is zero, the proxy is free to pull as many bytes as
available from the output ring buffer for sending them in one go to the
target file. Conversely, any non-zero granularity value is used as the
exact count of bytes which is written to the destination file by the
inband worker at each transfer. For instance, in the
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html) use case,
we would use sizeof(uint64_t). You may pass zero for a memory mapping
proxy since no granularity is applicable in this case.
{{% /argument %}}

{{% argument fmt %}}
A [printf(3)](http://man7.org/linux/man-pages/man3/printf.3.html)-like
format string to generate the proxy name. A common way of generating
unique names is to add the calling process's _pid_ somewhere into the
format string as illustrated in the example. The generated name is
used to form a last part of a pathname, referring to the new
[proxy]({{< relref "core/_index.md" >}}) device in the file system. So
this name must contain only valid characters in this context,
excluding slashes.
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

`evl_new_proxy()` returns the file descriptor of the new proxy on
success. If the call fails, a negated error code is returned instead:

- -EEXIST	The generated name is conflicting with an existing proxy.

- -EINVAL	Either _bufsz_ and/or _granularity_ are wrong, or the
  		generated proxy name is badly formed, likely containing
		invalid character(s), such as a slash. Keep in mind that
		it should be usable as a basename of a device file path.
		_bufsz_ must not exceed 2^30. If _granularity_ is non-zero,
		_bufsz_ must be a multiple of this value.

- -EBADF	_targetfd_ is not a valid file descriptor.

- -ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

- -ENXIO        The EVL library is not initialized for the current process.
  		Such initialization happens implicitly when `evl_attach_self()`
		is called by any thread of your process, or by explicitly
		calling `evl_init()`. You have to bootstrap the library
		services in a way or another before creating an EVL proxy.

---

{{< proto evl_send_proxy >}}
ssize_t evl_send_proxy(int proxyfd, const void *buf, size_t count)
{{< /proto >}}

This is a shorthand checking the current execution stage before
sending the output through a proxy channel via the proper system call,
i.e. [write(2)](http://man7.org/linux/man-pages/man2/write.2.html) if
inband, or [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) if out-of-band. You can
use this routine to emit output from a portion of code which may be
used from both stages.

{{% argument proxyfd %}}
A descriptor referring to the proxy which should handle the output data.
{{% /argument %}}

{{% argument buf %}}
A buffer containing the data to be written.
{{% /argument %}}

{{% argument count %}}
The number of bytes to write starting from _buf_.
Zero is acceptable and leads to a null-effect.
{{% /argument %}}

`evl_send_proxy()` returns the number of bytes sent through the proxy. A
negated error code is returned on failure, which corresponds to the
_errno_ value received from either
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) or
[oob_write()]({{< relref "core/user-api/io/_index.md#oob_write" >}})
depending on the calling stage.

---

{{< proto evl_print_proxy >}}
int evl_vprint_proxy(int proxyfd, const char *fmt, ...)
{{< /proto >}}

A routine which formats a
[printf(3)](http://man7.org/linux/man-pages/man3/printf.3.html)-like
input string before sending the resulting output through a proxy
channel.

{{% argument proxyfd %}}
A descriptor referring to the proxy which should handle the output data.
{{% /argument %}}

{{% argument fmt %}}
The format string.
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

`evl_print_proxy()` returns the number of bytes sent through the
proxy. A negated error code is returned on failure, which may
correspond to either a formatting error, or to a sending error in
which case the error codes returned by [evl_send_proxy()]({{< relref
"#evl_send_proxy" >}}) apply.

---

{{< proto evl_vprint_proxy >}}
int evl_vprint_proxy(int proxyfd, const char *fmt, va_list ap)
{{< /proto >}}

This call is a variant of [evl_print_proxy()]({{< relref
"#evl_print_proxy" >}}) which accepts format arguments specified as a
pointer to a variadic parameter list.

{{% argument proxyfd %}}
A descriptor referring to the proxy which should handle the output data.
{{% /argument %}}

{{% argument fmt %}}
The format string.
{{% /argument %}}

{{% argument ap %}}
A pointer to a variadic parameter list.
{{% /argument %}}

`evl_vprint_proxy()` returns the number of bytes sent through the
proxy. A negated error code is returned on failure, which may
correspond to either a formatting error, or to a sending error in
which case the error codes returned by [evl_send_proxy()]({{< relref
"#evl_send_proxy" >}}) apply.

---

{{< proto evl_printf >}}
int evl_printf(const char *fmt, ...)
{{< /proto >}}

A shorthand which sends formatted output through an internal proxy
targeting _fileno(stdout)_. This particular proxy is built by the EVL
library automatically when it initializes as a result of a direct or
indirect call to `evl_init()`.

---

### Events pollable from a proxy file descriptor

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}})
interface can monitor the following events occurring on a proxy file
descriptor:

- _POLLOUT_ and _POLLWRNORM_ are set whenever the output ring buffer
  the file descriptor refers to is empty AND allocated, which means
  that a proxy initialized with a zero-sized output buffer never
  raises these events.
