---
title: "Using libevl"
menuTitle: "Application interface"
weight: 5
pre: "&rsaquo; "
---

## What is an "EVL application process"?

An EVL application process is composed of one or more EVL threads,
running along with any number of regular POSIX threads.

- an EVL thread is a regular POSIX thread (spawned by a call to
`pthread_create()` or the `main()` context) which has issued the
[evl_attach_self()]({{< relref
"core/user-api/thread/_index.md#evl_attach_thread" >}}) system
call. This service binds the caller to the EVL core, which enables it
to invoke the real-time, ultra-low latency services the latter
delivers.

- once attached, such thread may call EVL core services, until it
detaches from the core by a call to [evl_detach_self()]({{< relref
"core/user-api/thread/_index.md#evl_detach_thread" >}}), or exits,
whichever comes first. It may still call routines from the standard
\*libc, except when real-time guarantees are required.

- when an EVL thread requires real-time guarantees, it must use the
proper services provided by `libevl` exclusively. If it calls a
regular \*libc service while in the middle of a time-critical code,
willingly or mistakenly, the EVL core will keep the system safe by
transparently demoting the caller to [in-band context]({{< relref
"dovetail/altsched/_index.md#altsched-theory" >}}) until the latter
calls a real-time EVL service again.  However, the calling thread
obviously lost any real-time guarantee in the process.

To sum up, the lifetime of an EVL thread usually looks like this:

1. When starting, an EVL thread runs the \*libc services it may need in
order to setup/prepare for the time-critical work loop, like opening
files, allocating system memory, creating EVL objects with
evl\_new\_*() calls, and so on. At some point, it must call
[evl_attach_self()]({{< relref
"core/user-api/thread/_index.md#evl_attach_thread" >}}) in order to
attach to the EVL core.

2. The EVL thread runs its time-critical work loop, only calling EVL
services which operate from the [out-of-band context]({{< relref
"dovetail/altsched/_index.md#altsched-theory" >}}), therefore
guaranteeing bounded, ultra-low latency. The pivotal EVL service from
such loop has to be a blocking call, waiting for the next real-time
event to process. For instance, such call could be
[evl_wait_flags()]({{< relref
"core/user-api/flags/_index.md#evl_wait_flags" >}}),
[evl_get_sem()]({{< relref
"core/user-api/semaphore/_index.md#evl_get_sem" >}}), [evl_poll()]({{<
relref "core/user-api/poll/_index.md#evl_poll" >}}), [oob_read()]({{<
relref "core/user-api/io/_index.md#oob_read" >}}) and so on.

3. Eventually, the EVL thread may call regular \*libc services in order
to cleanup/unwind the application context when its time-critical loop
is over and the thread/app wants to exit.

[This page]({{< relref "core/user-api/function_index/_index.md" >}})
is an index of all EVL system calls available to applications, which
should help you finding out which call is legit from which context.
In order to use the ultra-low latency EVL services, you need to link
your application code against the [libevl]({{< relref
"core/build-steps.md#building-libevl" >}}) library which provides the
EVL system call wrappers.

## Is libevl a replacement for my favourite \*libc? (glibc, musl, uClibc...)

_NO, not even remotely._ This is a drop-in _complement_ to the regular
C library and NPTL support you may be using, which enables your
thread(s) of choice to be scheduled with ultra-low latency guarantee
by the EVL core. As it should be clear now from the above section, you
may - and actually have to - use a combination of these libraries into
a single application, but you must do this in a way that ensures that
your time-critical code only relies on either:

- the [well-defined set]({{< relref
  "core/user-api/function_index/_index.md" >}}) of `libevl`'s
  low-latency services which operates from the [out-of-band
  context]({{< relref "dovetail/altsched/_index.md#altsched-theory"
  >}}).

- a (very) small subset of the \*libc calls which are known _not to_
  depend on regular in-band kernel services. In other words, routines
  which won't issue common Linux system calls in any way. For
  instance, routines from the string(3) and memcpy(3) sections come to
  mind here, like strcpy(), memcpy() and friends. At the opposite, any
  call which directly or indirectly might call malloc(3) must be
  banned from your time-critical code (think about most stdio(3)
  calls, C++ default constructors etc).

Outside of those time-critical sections which require the EVL core to
guarantee ultra-low latency scheduling for your application threads,
your code may happily call whatever service from whatever \*libc.
