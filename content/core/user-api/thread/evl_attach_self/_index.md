---
title: "Attach a thread"
menuTitle: "evl_attach_self"
weight: 25
---

### int evl_attach_self(const char *_fmt_, ...)

EVL does not really _create_ threads; instead, it enables a regular
POSIX thread to invoke its real-time services once this thread has
attached to the EVL core.  `evl_attach_self()` is the library call
which requests such attachment. All you need to provide is a _unique_
thread name, which will be the name of the device element representing
that thread in the file system.

There is no requirement as of when `evl_attach_self()` should be
called in a thread's execution flow. You just have to call it before
it starts requesting EVL services.

{{% notice note %}}
The `main()` thread of a process is no different from any other thread
to EVL. It may call `evl_attach_self()` whenever you see fit, or not
at all if you don't plan to request EVL services from this context.
{{% /notice %}}

#### _fmt_

A printf-like format string to generate the thread name. A common way
of generating unique thread names is to add the calling process's
_pid_ somewhere into the format string as illustrated in the example.

{{% notice warning %}}
The generated name is used to form a file path, referring to the new
thread element's device in the file system. So this name must contain
only valid characters in this context, excluding slashes.
{{% /notice %}}

#### ...

The optional variable argument list completing the format.

### Return value

`evl_attach_self()` returns the file descriptor of the newly attached
thread on success. You may use this _fd_ to submit requests for the
newly attached thread in other calls from the EVL library which ask
for a thread file descriptor.

If the call fails, a negated error code is returned:

-EEXIST		The generated name is conflicting with an existing thread's
		name.

-EINVAL		The generated name is badly formed, likely containing
		invalid character(s), such as a slash. Keep in mind that
		it should be usable as a basename of a device element's file path.

-ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

-EMFILE		The per-process limit on the number of open file descriptors
		has been reached.

-ENFILE		The system-wide limit on the total number of open files
		has been reached.

-EPERM		The caller is not allowed to lock memory via a call to
		`mlockall(2)`. Since memory locking is a requirement for running
		EVL threads, no joy.

-ENOMEM		No memory available, whether the kernel could not
		lock down all of the calling process's virtual address
		space into RAM, or some other reason related to some
		process or driver eating way too much virtual or physical
		memory.	You may start panicking.

-ENOSYS		There is no EVL core enabled in the running kernel.

Anything else is likely an EVL bug. Congratulations.

### Example

```
#include <evenless/thread.h>

static void *byte_crunching_thread(void *arg)
{
	int efd;

	/* Attach the current thread to the EVL core. */
	efd = evl_attach_self("cruncher-%d", getpid());
	...
}
```

As a result of this call, you should see a new device appear into the
_/dev/evenless/thread_ hierarchy, e.g.:

```
$ ls -l /dev/evenless/threads
total 0
crw-rw----    1 root     root      246,   1 Jan  1  1970 clone
crw-rw----    1 root     root      246,   1 Jan  1  1970 cruncher-2712
```

### Notes

1. You can revert the attachment to EVL at any time by calling
[`evl_detach_self()`]({{% relref
"core/user-api/thread/evl_detach_self/_index.md" %}}) from the context
of the thread to detach.

2. Closing all the file descriptors referring to an EVL thread is not
enough to drop its attachment to the EVL core. It merely prevents to
submit any further request for the original thread via calls taking
file descriptors. You would still have to call
[`evl_detach_self()`]({{% relref
"core/user-api/thread/evl_detach_self/_index.md" %}}) from the context
of this thread to fully detach it.

3. If a valid file descriptor is still referring to a detached thread,
or after the thread has exited, any request submitted for that thread
using such _fd_ would receive -ESTALE.

4. An EVL thread which exits is automatically detached from the EVL
core, you don't have to call [`evl_detach_self()`]({{% relref
"core/user-api/thread/evl_detach_self/_index.md" %}}) explicitly
before exiting your thread.

5. The EVL core drops the kernel resources attached to a thread once
it has detached itself or has exited, and only after all the file
descriptors referring to that thread have been closed.

6. The EVL library sets the O_CLOEXEC flag on the file descriptor
referring to the newly attached thread before returning from
`evl_attach_self()`.
