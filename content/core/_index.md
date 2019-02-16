---
title: "Real-time core"
menuTitle: "Real-time core"
date: 2019-02-16T16:10:44+01:00
weight: 5
pre: "<b>1. </b>"
---

## Make it ordinary

The idea is to build EVL as an ordinary feature of the Linux kernel,
not as a foreign extension slapped on top of it.  [Dovetail]({{%
relref "dovetail/_index.md" %}}) plays an important part here, as it
hides the nitty-gritty details of embedding a companion co-kernel into
Linux from us.

## Make it simple

The user-space interface to this core is the EVL library
(`libevenless.so`), which implements the system call interface, along
with the fundamental thread synchronization services. No bells and
whistles, only the basic stuff. The intent is to provide simple
mechanisms, complex semantics and policies can and should be
implemented in high level APIs based on this library.

## Elements

As the name suggests, _elements_ are the basic features we may require
from the EVL core for supporting real-time applications in this dual
kernel environment. Also, these features have to implemented as
kernel-based resources for delivering the service; at the opposite,
pure user-space code could not deliver it. So far, it looks like we
need only four elements:

- thread. As the basic execution unit, we want it to be runnable
  either in real-time mode or regular GPOS mode alternatively, which
  exactly maps to [Dovetail's]({{% relref
  "dovetail/pipeline/_index.md" %}}) out-of-band and inband contexts.

- monitor. This element has the same purpose than the main kernel's
  _futex_, which is about providing an integrated - although much
  simpler - set of fundamental thread synchronization features. It is
  used by the EVL library to implement mutexes, events
  (i.e. _condition variables_, sort of) and semaphores in user-space.

- clock. We may find platform-specific clock devices in addition to
  the core ones defined by the architecture, for which ad hoc drivers
  should be written. The clock element ensures that all clock drivers
  present the same interface to applications in user-space. In
  addition, the clock element is able to export individual timers
  to applications.

- cross-buffer. A cross-buffer (aka _xbuf_) is a communication channel
  we need to exchange data between the real-time and common
  contexts. Unlike a file proxy, it is bi-directional and any kind of
  threads (EVL or regular) should be allowed to wait for input from
  the other side. This should be an equivalent to Xenomai's _XDDP_
  socket protocol (aka _message pipes_).

## Everything is a file

The nice thing about the file semantics is that it may solve several
problems for our embedded co-kernel:

- it can organize resource management for our kernel objects. If every
  element we create is represented by a file, we can leave the hard
  work of managing the creation and release process to the
  [VFS](https://www.kernel.org/doc/Documentation/filesystems/vfs.txt),
  tracking references to every element from file descriptors.

- if a file backing some element can be obtained by opening a device
  present in the file system, we are done with providing applications
  a way to share this element between multiple processes: these
  processes would only need to open the same device file for sharing
  the underlying element.

- we can benefit from the file permission, monitoring and auditing
  logic attached to files for our own elements.

Now, one might wonder: since the main kernel would be involved in
creating and deleting elements, wouldn't this prevent us from doing so
in mere real-time mode? Short answer: surely it would, and this is
just fine. Nearly two decades after [Xenomai](https://xenomai.org/)
v1, I'm still to find the first practical use case which would require
this. As a matter of fact, those potentially heavyweight operations
can and should happen when the application is not busy running
time-critical code.

The above translates as follows in EVL:

- Each time an application creates a new element, a character device
  appears in the file system hierarchy under a directory named
  /dev/evenless/*element_type*/. By opening the device file, the
  application receives a file descriptor which can be used for
  controlling and/or exchanging data with the underlying element. This
  is definitely a regular file descriptor, on a regular character
  device.

- Since every element is backed by a kernel device, we may also bind
  _udev_ rules to events of interest on such element. We may also
  export the internal state of any element via the /sysfs, which is
  much better than stuffing /proc with even more ad hoc files for the
  same purpose.

- Since every file opened on the same device refers to the same EVL
  element, we have our handle for sharing elements between processes.

- Even local resources created by the EVL core passed to applications
  which are not elements are also backed by a file, like clock-based
  individual timers.

- Using file descriptors, the application can monitor _events_
  occurring on an arbitrary set of elements with a single EVL system
  call, just like one would do using `poll(2)`, `epoll(7)` or
  `select(2)`.

## Basic utilities

We also need EVL to provide a few ancillary services, so that high
level APIs or even applications could build over them. So far, the
following came to mind:

- synchronous I/O multiplexing. Since everything is a file in EVL, and
  we have file descriptors to play with in applications for referring
  to them, providing the EVL equivalent of `[e]poll` just makes sense.

- file proxy. Linux-based dual kernel systems are nasty by design: the
  huge set of GPOS features is always visible to applications but they
  are not allowed to use it when they carry out real-time work with
  the help of the co-kernel, such as manipulating files created by the
  main kernel.  So much for using `printf(3)` in some time-sensitive
  loop in this case. The file proxy should help in easing the pain, by
  channeling file output through the _vfs_ in a way that keeps the
  real-time side happy. Although the proxy is technically part of the
  element framework in EVL, applications could resort to a pure
  user-space implementation for roughly the same purpose, even if in a
  much more convoluted way.  So the proxy is more like a utility.

- trace channel. We need a way to emit traces from applications
  through the main kernel's FTRACE mechanism for debugging purpose,
  directly from the real-time context. Dovetail readily enables
  tracing from the out-of-band stage, so all we need is an interface
  here.

{{% notice note %}}
We cannot use the file proxy to relay the traces through the
`trace_marker` file, because we want the tracepoint to appear in the
output stream at the exact time the code running [out-of-band]({{%
relref "dovetail/pipeline/_index.md" %}}) issued it. On the contrary,
channeling the trace data through the proxy would mean to defer the
trace output by design, until the main kernel resumes execution.
{{% /notice %}}

## EVL device drivers are (almost) common drivers

EVL does not introduce any specific driver model. It exports a
dedicated [kernel API]({{%relref "core/kernel-api/_index.md" %}}) for
implementing real-time I/O operations in common character device
drivers. In fact, the EVL core is composed of a set of such drivers,
implementing each class of elements and utilities.
