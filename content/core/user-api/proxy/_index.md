---
title: "File proxy"
weight: 40
---

### Zero-latency I/O to in-band files

{{% mixedgrid-right src="/images/proxy.png" %}}
A common issue with dual kernel systems stems from the requirement
_not_ to issue [in-band system calls]({{< relref
"dovetail/altsched.md#inband-switch" >}}) while running time-critical
code in out-of-band context. Problem is that sometimes, you may need -
for instance - to write to regular files such as for logging
information to the console or elsewhere from an out-of-band work
loop. Doing so by calling common
[stdio(3)](http://man7.org/linux/man-pages/man3/stdio.3.html) routines
directly is therefore not an option for latency reasons.

The EVL core solves this problem with the _file proxy_ feature, which
can push data to an arbitrary target file, and/or conversely read data
pulled from such target file, providing an [out-of-band I/O
interface]({{< relref "core/user-api/io/_index.md" >}}) to the data
producer and consumer threads. This works by offloading the I/O
transfers to internal worker threads running in-band which relay the
data to/from the target file. For this reason, I/O transfers may be
delayed until the in-band stage resumes on a worker's CPU.  For
instance, [evl_printf()]({{< relref "#evl_printf" >}}) formats then
emits the resulting string to _fileno(stdout)_ via an internal proxy
created by `libevl.so` at [initialization]({{< relref
"core/user-api/init/_index.md" >}}).
{{% /mixedgrid-right %}}

{{% mixedgrid src="/images/proxy-read.png" %}}
Typically, out-of-band writers would send data through the proxy file
descriptor using EVL's [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) system call, which in-band
readers could receive using the
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html) system
call. Conversely, out-of-band readers would receive data through the
proxy file descriptor using EVL's [oob_read()]({{< relref
"core/user-api/io/_index.md#oob_read" >}}) system call, which in-band
writers could send using the
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) system
call. You can associate any type of file with a proxy, including a
[socket(2)](http://man7.org/linux/man-pages/man2/socket.2.html),
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html),
[signalfd(2)](http://man7.org/linux/man-pages/man2/signalfd.2.html),
[pipe(2)](http://man7.org/linux/man-pages/man7/pipe.7.html) and so on
(see also the discussion below about the transfer [granularity]({{<
relref "#evl_create_proxy" >}})). This means that you could also use a
proxy for signaling an in-band object from the out-of-band context, if
doing so can be done using a regular
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) system
call, like
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html) and
[signalfd(2)](http://man7.org/linux/man-pages/man2/signalfd.2.html).

The proxy also handles input and output operations from callers
running on the in-band stage transparently.
{{% /mixedgrid %}}

{{% notice warning %}}
Starting from [kernel v5.10](https://lkml.org/lkml/2020/9/3/555), a
restricted set of file types supports proxying compared to earlier
releases, specifically those for which the corresponding driver
implements the `read_iter()`, `write_iter()` file operations. Regular
files, sockets, pipes, tty and eventfd can still be used as proxy
targets though.
{{% /notice %}}

#### Logging debug messages via a proxy {#proxy-logging-example}

For instance, you may want your application to dump debug information
to some arbitray file as it runs, including when running time-critical
code out-of-band. Admittedly, this would add some overhead, but still
low enough to keep the system happy, while giving you precious debug
hints. Obviously, you cannot get away with calling the plain
[printf(3)](http://man7.org/linux/man-pages/man3/printf.3.html)
service for this, since it would downgrade the calling EVL thread to
[in-band mode]({{< relref "dovetail/altsched.md#inband-switch"
>}}). However, you may create a proxy to the debug file:

```
	#include <fcntl.h>
	#include <evl/proxy.h>

	int proxyfd, debugfd;

	debugfd = open("/tmp/debug.log", O_WRONLY|O_CREAT|O_TRUNC, 0600);
	...
	/* Create a proxy offering a 1 MB buffer. */
	proxyfd = evl_new_proxy(debugfd, 1024*1024, "log:%d", getpid());
	...
```

Then channel debug output from anywhere in your code you see fit
through the proxy this way:

```
	#include <evl/proxy.h>

	evl_print_proxy(proxyfd, "some useful debug information");
```

#### Synchronizing in-band threads on out-of-band events with via a proxy {#proxy-synchronization-example}

There are a couple of ways which you could use in order to wake up an
in-band thread waiting for some event to occur on the out-of-band side
of your application, while preventing the waker from being demoted as
a result of sending such a signal. One would involve running the
in-band thread in the [SCHED_WEAK scheduling class]({{< relref
"core/user-api/scheduling/_index.md#SCHED_WEAK" >}}), waiting on some
EVL synchronization object, such as a [semaphore]({{< relref
"core/user-api/semaphore/_index.md" >}}) or [event flag group]({{<
relref "core/user-api/event/_index.md" >}}). Another one would use a
[cross-buffer]({{< relref "core/user-api/xbuf/_index.md" >}}) for
sending some wakeup datagram from the out-of-band caller to the in-band
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
	/* Create a private proxy for output, allow up to 3 notifications to pile up. */
	proxyfd = evl_create_proxy(evntfd, sizeof(uint64_t) * 3, sizeof(uint64_t), 0, "event-relay");
	...
```

Note the **specific granularity value** mentioned in the creation call
above: we do want the proxy to write 64-bit values at each transfer,
in order to cope with the requirements of the
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html)
interface. Once the proxy is created, set up for routing all write
requests to the regular event file descriptor by 64-bit chunks, the
in-band thread can wait for out-of-band events reading from the other
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
		ssize_t ret;
		uint64 val;
		
		for (;;) {
		    /* Wait for the next wakeup signal. */
		    ret = read(evntfd, &val, sizeof(val));
		    ...
		}
	}
```

### Export of memory mappings {#proxy-mapping-export}

EVL proxies can be also be used for carrying over memory mapping
requests to a final device which actually serves the mapping.  The
issuer of such request does not have to know which device driver will
actually be handling that request: the proxy acts as an anchor point
agreed between peer processes in order to locate a particular
memory-mappable resource, and the proxy simply redirects the mapping
requests it receives to the device driver handling the target file it
is proxying for.

![Alt text](/images/proxy-mmap.png "Mapping proxy")

#### Exporting process-private memfd memory to the outer world {#proxy-memfd-example}

For instance, this usage of a public proxy comes in handy when you
need to export a private memory mapping like those obtained by
[memfd_create(2)](http://man7.org/linux/man-pages/man2/memfd_create.2.html)
to a peer process, assuming you don't want to deal with the hassle of
the POSIX
[shm_open(3)](http://man7.org/linux/man-pages/man3/shm_open.3.html)
interface for sharing memory objects.  In this case, all this peer has
to know is the name of the proxy which is associated with the
memory-mappable device. It can
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html) that proxy
device in [/dev/evl/proxy]({{< relref
"core/user-api/_index.md#evl-fs-hierarchy" >}}), then issue the
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
	/* Create a public proxy others can find in the /dev/evl/proxy hierarchy. */
	proxyfd = evl_new_proxy(memfd, 0, "/some-fancy-proxy");
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

The other way you could do this is by passing the _memfd_ file
descriptor via a socket-based control message, but this would require
significantly more logic in both peers than using a proxy.

### Proxy services {#proxy-services}

{{< proto evl_create_proxy >}}
int evl_create_proxy(int targetfd, size_t bufsz, size_t granularity, int flags, const char *fmt, ...)
{{< /proto >}}

This call creates a new proxy to the target file referred to by
_targetfd_, then returns a file descriptor for accessing the proxy
upon success. [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) should be used to send
data through the proxy to the target file. Conversely,
[oob_read()]({{< relref "core/user-api/io/_index.md#oob_read" >}})
should be used to receive data through the proxy from the target file.

{{% argument targetfd %}}
A file descriptor referring to the target file which should
be associated with the proxy.
{{% /argument %}}

{{% argument bufsz %}}
The size in bytes of the I/O ring buffer where the relayed data is
kept.  _bufsz_ must not exceed 2^30. A ring is allocated for each
direction enabled in _flags_ (see `EVL_CLONE_OUTPUT`,
`EVL_CLONE_INPUT`). If _granularity_ is non-zero, _bufsz_ must be a
multiple of this value. Out-of-band readers/writers may block on an
out-of-band file operation if the buffer is either full on output or
empty on input, unless the proxy descriptor is set to [non-blocking
mode
(O_NONBLOCK)](http://man7.org/linux/man-pages/man2/fcntl.2.html). Zero
is an acceptable value if you plan to use this proxy exclusively for
exporting a shared memory mapping, in which case there is no point in
reserving I/O ring buffers.
{{% /argument %}}

{{% argument granularity %}}
In some cases, the target file may have special semantics, which
requires a fixed amount of data to be submitted at each read/write
operation, like the
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html) file
which requires 64-bit words to be read or written from/to it at each
transfer. When granularity is less than 2, the proxy is free to read
or write as many bytes as possible from/to the target file at each
transfer performed by the worker. Conversely, a granularity value
greater than 1 is used as the exact count of bytes which may be read
from or written to the target file by the in-band worker at each
transfer. For instance, in the
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html) use
case, we would use sizeof(uint64_t). You may pass zero for a memory
mapping proxy since no granularity is applicable in this case.
{{% /argument %}}

{{% argument flags %}}
A set of creation flags for the new element, defining how it should be
operated, along with its [visibility]({{< relref
"core/user-api/_index.md#element-visibility" >}}):

  - `EVL_CLONE_OUTPUT` enables the output side of the proxy, so that
    writing to the proxy file descriptor pushes data to _targetfd_. If
    _busfz_ is non-zero and _flags_ does not mention
    `EVL_CLONE_INPUT`, `EVL_CLONE_OUTPUT` is assumed (default
    behavior).

  - `EVL_CLONE_INPUT` enables the input side of the proxy, so that
    reading from the proxy file descriptor pulls data from _targetfd_.

  - `EVL_CLONE_PUBLIC` denotes a public element which is represented
    by a device file in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file
    hierarchy, which makes it visible to other processes for sharing.
  
  - `EVL_CLONE_PRIVATE` denotes an element which is private to the
    calling process. No device file appears for it in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy.

  - `EVL_CLONE_NONBLOCK` sets the file descriptor of the new proxy in
    non-blocking I/O mode (`O_NONBLOCK`). By default, `O_NONBLOCK` is
    cleared for the file descriptor.
 {{% /argument %}}

{{% argument fmt %}}
A [printf](http://man7.org/linux/man-pages/man3/printf.3.html)-like
format string to generate the proxy name. See this description of the
[naming convention]
({{< relref "core/user-api/_index.md#element-naming-convention" >}}).
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_create_proxy()]({{< relref "#evl_create_proxy" >}}) returns the
file descriptor of the new proxy on success. If the call fails, a
negated error code is returned instead:

- -EEXIST	The generated name is conflicting with an existing proxy name.

- -EINVAL	Either _flags_ is wrong,  _bufsz_ and/or _granularity_ are wrong,
  		or the [generated name]({{< relref "core/user-api/_index.md#element-naming-convention"
  		>}}) is badly formed.

- -EINVAL	_bufsz_ is zero but _flags_ mentions any of `EVL_CLONE_INPUT`
  		or `EVL_CLONE_OUTPUT`.

- -EBADF	_targetfd_ is not a valid file descriptor.

- -ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

- -ENXIO        The EVL library is not initialized for the current process.
  		Such initialization happens implicitly when [evl_attach_self()]({{% relref
  		"core/user-api/thread/_index.md#evl_attach_self" %}})
		is called by any thread of your process, or by explicitly
		calling [evl_init()]({{<
  		relref "core/user-api/init/_index.md#evl_init"
  		>}}). You have to bootstrap the library
		services in a way or another before creating an EVL proxy.

---

{{< proto evl_new_proxy >}}
int evl_new_proxy(int targetfd, size_t bufsz, const char *fmt, ...)
{{< /proto >}}

This call is a shorthand for creating a private proxy with no particular
transfer granularity. It is identical to calling:

```
	evl_create_proxy(targetfd, bufsz, 0, EVL_CLONE_PRIVATE, fmt, ...);
```

If _bufsz_ is non-zero, the output side of the proxy is implicitly
enabled. If _busfz_ is zero, then a mapping-only proxy is created.

{{% notice info %}}
Note that if the [generated name] ({{< relref
"core/user-api/_index.md#element-naming-convention" >}}) starts with a
slash ('/') character, `EVL_CLONE_PRIVATE` would be automatically turned
into `EVL_CLONE_PUBLIC` internally.
{{% /notice %}}

---

{{< proto evl_send_proxy >}}
ssize_t evl_send_proxy(int proxyfd, const void *buf, size_t count)
{{< /proto >}}

This is a shorthand checking the current execution stage before
sending the output through a proxy channel via the proper system call,
i.e. [write(2)](http://man7.org/linux/man-pages/man2/write.2.html) if
in-band, or [oob_write()]({{< relref
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

[evl_send_proxy()]({{< relref "#evl_send_proxy" >}}) returns the
number of bytes sent through the proxy. A negated error code is
returned on failure, which corresponds to the _errno_ value received
from either
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

[evl_print_proxy()]({{< relref "#evl_print_proxy" >}}) returns the
number of bytes sent through the proxy. A negated error code is
returned on failure, which may correspond to either a formatting
error, or to a sending error in which case the error codes returned by
[evl_send_proxy()]({{< relref "#evl_send_proxy" >}}) apply.

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

[evl_vprint_proxy()]({{< relref "#evl_vprint_proxy" >}}) returns the
number of bytes sent through the proxy. A negated error code is
returned on failure, which may correspond to either a formatting
error, or to a sending error in which case the error codes returned by
[evl_send_proxy()]({{< relref "#evl_send_proxy" >}}) apply.

---

{{< proto evl_printf >}}
int evl_printf(const char *fmt, ...)
{{< /proto >}}

A shorthand which sends formatted output through an internal proxy
targeting _fileno(stdout)_. This particular proxy is built by the EVL
library automatically when it initializes as a result of a direct or
indirect call to [evl_init()]({{< relref
"core/user-api/init/_index.md#evl_init" >}}).

---

### Events pollable from a proxy file descriptor {#proxy-poll-events}

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}})
interface can monitor the following events occurring on a proxy file
descriptor:

- `POLLOUT` and `POLLWRNORM` are set whenever the output ring buffer
  some polled file descriptor refers to is empty AND allocated, which
  means that a proxy initialized with a zero-sized output buffer never
  raises these events. You would typically monitor the `POLLOUT`
  condition in order to wait for all of the buffered output to have
  been sent to the target file.

- `POLLIN` and `POLLRDNORM` are set whenever some polled file
  descriptor refers to a proxy for which data is available on input.

- `POLLERR` is returned whenever some polled file descriptor refers to
  a proxy which may not be either read or written (see
  `EVL_CLONE_OUTPUT`, `EVL_CLONE_INPUT` from ({{< relref
  "#evl_create_proxy" >}})).

---

{{<lastmodified>}}
