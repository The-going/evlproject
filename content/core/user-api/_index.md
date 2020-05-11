---
title: "Using libevl"
menuTitle: "Application interface"
weight: 7
pre: "&#9702; "
---

## What is an "EVL application process"? {#evl-application}

An EVL application process is composed of one or more EVL threads,
running along with any number of regular POSIX threads.

- an EVL thread is initially a plain regular POSIX thread spawned by a call to
[pthread_create(3)](http://man7.org/linux/man-pages/man3/pthread_create.3.html)
or the `main()` context which has issued the [evl_attach_self()]({{<
relref "core/user-api/thread/_index.md#evl_attach_thread" >}}) system
call. This service binds the caller to the EVL core, which enables it
to invoke the real-time, ultra-low latency services the latter
delivers.

- once attached, such thread may call EVL core services, until it
detaches from the core by a call to [evl_detach_self()]({{< relref
"core/user-api/thread/_index.md#evl_detach_thread" >}}), or exits,
whichever comes first.

- whenever an EVL thread has to meet real-time requirements, it must
rely on the services provided by `libevl` exclusively. If such a
thread invokes a common C library service while in the middle of a
time-critical code, the EVL core does keep the system safe by
transparently demoting the caller to [in-band context]({{< relref
"dovetail/altsched.md#altsched-theory" >}}).  However, the calling
thread would loose any real-time guarantee in the process, meaning
that unwanted jitter would happen.

To sum up, the lifetime of an EVL application usually looks like this:

1. When a new process initializes, a regular thread - often the
`main()` one - invokes routines from the C library in order to get
common resources, like opening files, allocating memory buffers and so
on. At some point later on, this thread calls [evl_attach_self()]({{<
relref "core/user-api/thread/_index.md#evl_attach_thread" >}}) in
order to bind itself to the EVL core, which in turn allows it to
create other EVL objects (e.g. [evl_create_mutex()]({{< relref
"core/user-api/mutex/_index.md#evl_create_mutex" >}}),
[evl_create_event()]({{< relref
"core/user-api/event/_index.md#evl_create_event" >}}),
[evl_create_xbuf()]({{< relref
"core/user-api/xbuf/_index.md#evl_create_xbuf" >}})). If the application
needs more EVL threads, it simply spawns additional POSIX threads
using
[pthread_create(3)](http://man7.org/linux/man-pages/man3/pthread_create.3.html),
then ensures those threads bind themselves to the core with a call to
[evl_attach_self()]({{< relref
"core/user-api/thread/_index.md#evl_attach_thread" >}}).

2. Each EVL thread runs its time-critical work loop, only calling EVL
services which operate from the [out-of-band context]({{< relref
"dovetail/altsched.md#altsched-theory" >}}), therefore guaranteeing
bounded, ultra-low latency. The pivotal EVL service from such loop has
to be a blocking call, waiting for the next real-time event to
process. For instance, such call could be [evl_wait_flags()]({{<
relref "core/user-api/flags/_index.md#evl_wait_flags" >}}),
[evl_get_sem()]({{< relref
"core/user-api/semaphore/_index.md#evl_get_sem" >}}), [evl_poll()]({{<
relref "core/user-api/poll/_index.md#evl_poll" >}}), [oob_read()]({{<
relref "core/user-api/io/_index.md#oob_read" >}}) and so on.

3. Eventually, EVL threads may call common C library services in order
to cleanup/unwind the application context when their time-critical
loop is over and time has come to exit.

[This page]({{< relref "core/user-api/function_index/_index.md" >}})
is an index of all EVL system calls available to applications, which
should help you finding out which call is legit from which context.
In order to use the ultra-low latency EVL services, you need to link
your application code against the [libevl]({{< relref
"core/build-steps.md#building-libevl" >}}) library which provides the
EVL system call wrappers.

## Is libevl a replacement for my favourite C library? (glibc, musl, uClibc etc.)

_NO, not even remotely._ This is a drop-in _complement_ to the common
C library and NPTL support you may be using, which enables your
thread(s) of choice to be scheduled with ultra-low latency guarantee
by the EVL core. As it should be clear now from the above section, you
may - and actually have to - use a combination of these libraries into
a single application, but you must do this in a way that ensures
your time-critical code only relies on either:

- `libevl`'s [well-defined set]({{< relref
  "core/user-api/function_index/_index.md" >}}) of low-latency
  services which operate from the [out-of-band context]({{< relref
  "dovetail/altsched.md#altsched-theory" >}}).

- a (very) small subset of the common C library which are known _not_
  to depend on regular in-band kernel services. In other words, only
  routines which won't issue common Linux system calls in any way are
  fine in a time-critical execution context. For instance, routines
  from the
  [string(3)](http://man7.org/linux/man-pages/man3/string.3.html)
  section may be fine in a time-critical code, like
  [strcpy(3)](http://man7.org/linux/man-pages/man3/strcpy.3.html),
  [memcpy(3)](http://man7.org/linux/man-pages/man3/memcpy.3.html) and
  friends. At the opposite, any routine which may directly or
  indirectly invoke
  [malloc(3)](http://man7.org/linux/man-pages/man3/malloc.3.html) must
  be banned from your time-critical code, which includes
  [stdio(3)](http://man7.org/linux/man-pages/man3/stdio.3.html)
  routines, or C++ default constructors which rely on the standard
  memory allocator.

Outside of those time-critical sections which require the EVL core to
guarantee ultra-low latency scheduling for your application threads,
your code may happily call whatever service from whatever C library.

## What about multi-process applications? {#multi-process-apps}

`libevl` does not impose any policy regarding how you might want to
organize your application over multiple processes. However, the design
and implementation of the interface to the EVL core makes sharing EVL
resources a fairly simple task. EVL elements can be made visible as
common [devices]({{< relref "core/_index.md#everything-is-a-file"
>}}), such as [threads]({{< relref "core/user-api/thread/_index.md"
>}}), [mutexes]({{< relref "core/user-api/mutex/_index.md" >}}),
[events]({{< relref "core/user-api/event/_index.md" >}}),
[semaphores]({{< relref "core/user-api/semaphore/_index.md" >}}).
Therefore, every element you may want to share can be exported to the
device file system, based on a [visibility attribute]({{< relref
"#element-visibility" >}}) mentioned at creation time.

In addition, EVL provides a couple of additional features which come
in handy for sharing data between processes:

- a general memory-sharing mechanism based on the [file proxy]({{<
relref "core/user-api/proxy/_index.md#proxy-mapping-export" >}}),
which is used as an anchor point for memory-mappable devices.

- the [tube]({{< relref "core/user-api/tube/_index.md" >}}) data
structure, which is a lightweight FIFO working locklessly, you can
also use for [inter-process messaging]({{< relref
"core/user-api/tube/_index.md#inter-process-tube" >}}).

- the [Observable]({{< relref "core/user-api/observable/_index.md"
>}}) element which gives your application a built-in support for
implementing the [observer design
pattern](https://en.wikipedia.org/wiki/Observer_pattern) among one or
more processes transparently.

## Visibility: public and private elements {#element-visibility}

As hinted earlier, EVL elements created by the user API can be either
publically visible to other processes, or private to the process which
creates them. This is a choice you make at creation time, by passing
the proper visibility attribute to any of the `evl_create_*()` system
calls, either `EVL_CLONE_PUBLIC` or `EVL_CLONE_PRIVATE`.

A public element is represented in the [/dev/evl]({{< relref
"#evl-fs-hierarchy" >}}) hierarchy by a device file, which is visible
to any process.  Once a file descriptor is available from
[opening](http://man7.org/linux/man-pages/man2/open.2.html) such file,
it can be used to send requests to the element it refers to.

Conversely, a private element has no presence in the `/dev/evl`
hierarchy. Only the process which created such element receives a file
descriptor referring to it, directly from the creation call.

## The `/dev/evl` device file hierarchy {#evl-fs-hierarchy}

Because of its [everything-is-a-file]({{< relref
"core/_index.md#everything-is-a-file" >}}) mantra, EVL exports a
number of device files in the `/dev/evl` hierarchy, which lives in the
DEVTMPFS file system. Each device file either represents an active
[public element]({{< relref "#element-visibility" >}}), or a special
command device used internally by `libevl`. Element device files are
organized in directories, one for each [element class]({{< relref
"core/_index.md#evl-core-elements" >}}): clock, monitor, proxy, thread
and cross-buffer; general command devices appear at the top level,
such as _control_, _poll_ and _trace_:

```
~ # cd /dev/evl
/dev/evl # ls -l
total 0
drwxr-xr-x    2 root     root            80 Jan  1  1970 clock
crw-rw----    1 root     root      246,   0 Jan  1  1970 control
drwxr-xr-x    2 root     root            60 Apr 18 17:38 monitor
drwxr-xr-x    2 root     root            60 Apr 18 17:38 observable
crw-rw----    1 root     root      246,   3 Jan  1  1970 poll
drwxr-xr-x    2 root     root            60 Apr 18 17:38 proxy
drwxr-xr-x    2 root     root            60 Apr 18 17:38 thread
crw-rw----    1 root     root      246,   6 Jan  1  1970 trace
drwxr-xr-x    2 root     root            60 Apr 18 17:38 xbuf
```

Inside each class directory, the live public elements of that class
are visible, in addition to the special _clone_ command device. For
the curious, the role of this special device is documented in the
[under-the-hood]({{< relref
"core/under-the-hood/_index.md#hood-element-creation" >}}) section.

```
/dev/evl/thread # ls -l
total 0
crw-rw----    1 root     root      246,   1 Jan  1  1970 clone
crw-rw----    1 root     root      244,   0 Apr 19 10:45 timer-responder:2562
```

## Managing access permissions to EVL device files {#device-ownership-and-access}

In some situations, you may want to restrict access to EVL devices
files present in the [`/dev/evl`]({{< relref "#evl-fs-hierarchy" >}})
file hierarchy to a particular user or group of users. Because a
kernel device object is associated to each live EVL element in the
system, you can attach rules to UDEV events generated for public EVL
elements or special command devices appearing in the [`/dev/evl`]({{<
relref "#evl-fs-hierarchy" >}}) file hierarchy, in order to set up
their ownership and access permissions at creation time.

For [public elements]({{< relref "#element-visibility" >}}) which come
and go dynamically, EVL enforces a simple rule internally to set the
initial user and group ownership of any element device file, which is
to inherit it from the _clone_ device file of the class it belongs
to. For instance, if you set the ownership of the
`/dev/evl/thread/clone` device file via some UDEV rule to
`square.wheel`, all public [EVL threads]({{< relref
"core/user-api/thread/_index.md" >}}) will belong at creation time to
user `square`, group `wheel`.

## Element names {#element-naming-convention}

Every `evl_create_*()` call which creates a new element, along with
[evl_attach_thread]({{< relref
"core/user-api/thread/_index.md#evl_attach_thread" >}}) accepts a
[printf](http://man7.org/linux/man-pages/man3/printf.3.html)-like
format string to generate the element name. A common way of generating
unique names is to include the calling process's _pid_ somewhere into
the format string, so that you may start multiple instances of the
same application without running into naming conflicts. The
requirement for a unique name does not depend on the visibility
attribute: distinct elements must have distinct names, regardless of
their visibility. For instance:

```
#include <unistd.h>
#include <evl/thread.h>

	 ret = evl_attach_self("a-private-thread:%d", getpid());

~# ls -l /dev/evl/thread
total 0
crw-rw----    1 root     root      248,   1 Apr 17 11:59 clone
```

The generated name is used to create a `/sysfs` attribute directory
exporting the state information about the element. For [public
elements]({{< relref "#element-visibility" >}}), a device file is
created with the same name in the [/dev/evl]({{< relref
"#evl-fs-hierarchy" >}}) hierarchy, for accessing the element via the
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html) system
call.  Therefore, a name must contain only valid characters in the
context of a file name.

As a shorthand, `libevl` forces in the `EVL_CLONE_PUBLIC` [visibility
attribute]({{< relref "#element-visibility" >}}) whenever the element
name passed to the system call starts with a slash '/' character, in
which case this leading character will be skipped to form the actual
element name:

```
#include <unistd.h>
#include <evl/thread.h>

	 ret = evl_attach_self("/a-publically-visible-thread:%d", getpid());

~# ls -l /dev/evl/thread
total 0
crw-rw----    1 root     root      248,   1 Apr 17 11:59 clone
crw-rw----    1 root     root      246,   0 Apr 17 11:59 a-publically-visible-thread
```

Note that when found, such shorthand overrides the `EVL_CLONE_PRIVATE`
visibility attribute which might have been mentioned in the creation
flags for the same call. The slash character is invalid in any other
part of the element name, although it would be silently remapped to a
placeholder for private elements without leading to an API error.

---

{{<lastmodified>}}
