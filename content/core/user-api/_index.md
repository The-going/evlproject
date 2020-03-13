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
or the `main()` context) which has issued the [evl_attach_self()]({{<
relref "core/user-api/thread/_index.md#evl_attach_thread" >}}) system
call. This service binds the caller to the EVL core, which enables it
to invoke the real-time, ultra-low latency services the latter
delivers.

- once attached, such thread may call EVL core services, until it
detaches from the core by a call to [evl_detach_self()]({{< relref
"core/user-api/thread/_index.md#evl_detach_thread" >}}), or exits,
whichever comes first. It may still call routines from the common C
library such as _glibc_, except when real-time guarantees are
required.

- whenever an EVL thread requires real-time guarantees, it must use
the proper services provided by `libevl` exclusively. If such thread
invokes a common C library service while in the middle of a
time-critical code, the EVL core does keep the system safe by
transparently demoting the caller to [in-band context]({{< relref
"dovetail/altsched.md#altsched-theory" >}}).  However, the **calling
thread** obviously **lost any real-time guarantee** in the process.

To sum up, the lifetime of an EVL application usually looks like this:

1. When a new process initializes, a regular thread - often the
`main()` one - invokes routines from the C library in order to get
common resources, like opening files, allocating memory buffers and so
on. At some point later on, this thread calls [evl_attach_self()]({{<
relref "core/user-api/thread/_index.md#evl_attach_thread" >}}) in
order to bind itself to the EVL core, which in turn allows it to
create other EVL objects (e.g. [evl_new_mutex()]({{< relref
"core/user-api/mutex/_index.md#evl_new_mutex" >}}),
[evl_new_event()]({{< relref
"core/user-api/event/_index.md#evl_new_event" >}}),
[evl_new_xbuf()]({{< relref
"core/user-api/xbuf/_index.md#evl_new_xbuf" >}})). If the application
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

- a (very) small subset of the common C library which are known **NOT
  to** depend on regular in-band kernel services. In other words,
  routines which won't issue common Linux system calls in any way. For
  instance, routines from the
  [string(3)](http://man7.org/linux/man-pages/man3/string.3.html)
  section come to mind in this case, like
  [strcpy(3)](http://man7.org/linux/man-pages/man3/strcpy.3.html),
  [memcpy(3)](http://man7.org/linux/man-pages/man3/memcpy.3.html) and
  friends. At the opposite, any call which directly or indirectly
  might call
  [malloc(3)](http://man7.org/linux/man-pages/man3/malloc.3.html) must
  be banned from your time-critical code (think about most
  [stdio(3)](http://man7.org/linux/man-pages/man3/stdio.3.html) calls,
  C++ default constructors which rely on
  [malloc(3)](http://man7.org/linux/man-pages/man3/malloc.3.html)
  etc).

Outside of those time-critical sections which require the EVL core to
guarantee ultra-low latency scheduling for your application threads,
your code may happily call whatever service from whatever C library.

## What about multi-process applications?

`libevl` does not impose any policy regarding how you might want to
organize your application over multiple processes. However, the design
and implementation of the interface to the EVL core makes sharing EVL
resources a fairly simple task:

- most EVL resources are visible as common [devices in the DEVTMPFS
  file system]({{< relref "core/_index.md#everything-is-a-file"
  >}}). [Threads]({{< relref "core/user-api/thread/_index.md" >}}),
  [mutexes]({{< relref "core/user-api/mutex/_index.md" >}}),
  [events]({{< relref "core/user-api/event/_index.md" >}}),
  [semaphores]({{< relref "core/user-api/semaphore/_index.md" >}})
  etc., every resource you might want to share can be accessed by
  multiple processes as files.

- the implementation guarantees that any EVL resource which is
  accessible from the device file system can be safely manipulated by
  any process which is allowed to get a file descriptor on the
  corresponding device file. EVL system calls apply to the resource
  which is eventually referred to by the file descriptor. For
  instance, if a process may open a device file which path is
  _"/dev/evl/thread/supervisor"_, then it may send requests to the
  corresponding thread, regardless of the process it belongs to. Such
  thread was created by the [evl_attach_self("supervisor")]({{< relref
  "core/user-api/thread/_index.md#evl_thread_self" >}}) system call.

In addition, EVL provides a mechanism which come in handy for sharing
memory between processes, using the [file proxy]({{< relref
"core/user-api/proxy/_index.md#proxy-mapping-export" >}}) as an anchor
point for memory-mappable devices.

---

{{<lastmodified>}}
