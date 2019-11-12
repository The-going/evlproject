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
of this buffer to the target file represented by the destination
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
[socket](http://man7.org/linux/man-pages/man2/socket.2.html),
[eventfd](http://man7.org/linux/man-pages/man2/eventfd.2.html),
[signalfd](http://man7.org/linux/man-pages/man2/signalfd.2.html),
[pipe](http://man7.org/linux/man-pages/man7/pipe.7.html) and so on
(see also the discussion below about the write [granularity]({{<
relref "#evl_new_proxy" >}})). This means that you could also use a
proxy for signaling an inband object from the out-of-band context, if
doing so can be done using a regular
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html) system
call, like
[eventfd](http://man7.org/linux/man-pages/man2/eventfd.2.html) and
[signalfd](http://man7.org/linux/man-pages/man2/signalfd.2.html).

![Alt text](/images/proxy.png "File proxy")

### Export of memory mappings

EVL proxies can be also be used for carrying over memory mapping
requests to a final device which actually serves the mapping.  The
issuer of such request does not have to know which device driver will
be actually serving that request: the proxy acts as an anchor point
agreed between peer processes in order to locate a particular mappable
resource, and the proxy simply redirects the mapping requests it
receives to the device driver handling the target file it is proxying
for.

> Exporting private memfd memory to the outer world

For instance, this usage of a proxy comes in handy when you need to
export a private memory mapping like those obtained by
[memfd_create()](http://man7.org/linux/man-pages/man2/memfd_create.2.html)
to a peer process, assuming you don't want to deal with the hassle of
the POSIX
[shm_open(3)](http://man7.org/linux/man-pages/man3/shm_open.3.html)
interface for sharing memory objects.  in this case, all this peer has
to know is the name of the proxy which is associated with the mappable
device. It can
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html) that proxy
device in _/dev/evl/proxy/_, then issue the
[mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html) system
call for receiving a mapping to the backing device's memory. From the
application standpoint, creating then sharing a 1 KB RAM segment with
other peer processes may be as simple as this:

```
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

{{% argument targetfd %}}
A descriptor referring to the destination file which should receive
the output written to the proxy file descriptor returned by
`evl_new_proxy()`.
{{% /argument %}}

{{% argument bufsz %}}
The size in bytes of the circular buffer where the output data is kept
until the inband worker has relayed it to the destination file.
Out-of-band writers may block attempting to write to a proxy file
descriptor if the buffer is full, waiting to be depleted by the inband
worker thread, unless the proxy descriptor is set to non-blocking mode
(O_NONBLOCK).
{{% /argument %}}

{{% argument granularity %}}
In some cases, the target file may have special semantics, which
requires a fixed amount of data to be submitted at each write
operation, like the
[eventfd](http://man7.org/linux/man-pages/man2/eventfd.2.html) file
which requires 64-bit words to be written to it at each transfer. When
granularity is zero, the proxy is free to pull as many bytes as
available from the circular buffer for sending them in one go to the
target file. Conversely, any non-zero granularity value is used as the
exact count of bytes which is written to the destination file by the
inband worker at each transfer. For instance, in the
[eventfd](http://man7.org/linux/man-pages/man2/eventfd.2.html) case,
we would use sizeof(uint64_t).
{{% /argument %}}

{{% argument fmt %}}
A printf-like format string to generate the proxy name. A common
way of generating unique names is to add the calling process's _pid_
somewhere into the format string as illustrated in the example. The
generated name is used to form a last part of a pathname, referring to
the new [proxy]({{< relref "core/_index.md" >}}) device
in the file system. So this name must contain only valid characters in
this context, excluding slashes.
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
