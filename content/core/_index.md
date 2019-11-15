---
title: "The EVL core"
menuTitle: "Real-time core"
date: 2019-02-16T16:10:44+01:00
weight: 10
pre: "&#9656; "
---

## Make it ordinary, make it simple

The EVL core is an autonomous software core which is hosted by the
kernel, delivering real-time services to applications having to meet
stringent timing requirements. This small core is built like any
ordinary feature of the Linux kernel, not as a foreign extension
slapped on top of it.  [Dovetail]({{% relref "dovetail/_index.md" %}})
plays an important part here, as it hides the nitty-gritty details of
embedding a companion core into the kernel. Its fairly low code
footprint and limited complexity makes it a good choice as a
plug-and-forget real-time infrastructure, which can also be used as a
starting point for custom core implementations:

![Alt text](/images/kloc-core.png "EVL kernel code footprint")

The user-space interface to this core is the [EVL library]({{% relref
"core/user-api/_index.md" %}}) (`libevl.so`), which implements the
basic system call wrappers, along with the fundamental thread
synchronization services. No bells and whistles, only the basic
stuff. The intent is to provide simple mechanisms, complex semantics
and policies can and should be implemented in high level APIs based on
this library running in userland.

![Alt text](/images/kloc-user.png "EVL user code footprint")

## Elements {#evl-core-elements}

As the name suggests, _elements_ are the basic features we may require
from the EVL core for supporting real-time applications in this dual
kernel environment. Also, only the kernel could provide such features
in an efficient way, pure user-space code could not deliver. The EVL
core defines five of them:

- thread. As the basic execution unit, we want it to be runnable
  either in real-time mode or regular GPOS mode alternatively, which
  exactly maps to [Dovetail's]({{% relref
  "dovetail/pipeline/_index.md" %}}) out-of-band and in-band contexts.

- monitor. This element has the same purpose than the main kernel's
  _futex_, which is about providing an integrated - although much
  simpler - set of fundamental thread synchronization features. It is
  used by the EVL library to implement mutexes, condition variables,
  event flag groups and semaphores in user-space.

- clock. We may find platform-specific clock devices in addition to
  the core ones defined by the architecture, for which ad hoc drivers
  should be written. The clock element ensures that all clock drivers
  present the same interface to applications in user-space. In
  addition, this element can export individual software timers to
  applications which comes in handy for running periodic loops or
  waiting for oneshot events on a specific time base.

- cross-buffer. A cross-buffer (aka _xbuf_) is a bi-directional
  communication channel for exchanging data between out-of-band and
  in-band thread contexts, without impacting the real-time performance
  on the out-of-band side.  Any kind of thread (EVL or regular) can
  wait/poll for input from the other side. Cross-buffers serve the
  same purpose than Xenomai's _message pipes_ implemented by the
  _XDDP_ socket protocol.

- file proxy. Linux-based dual kernel systems are nasty by design: the
  huge set of GPOS features is always visible to applications but they
  should not to use it when they carry out real-time work with the
  help of the autonomous core, or risk unbounded response
  time. Because of such exclusion, manipulating files created by the
  main kernel such as calling
  [printf(3)](http://man7.org/linux/man-pages/man3/printf.3.html)
  should not be done directly from time-critical loops. A file proxy
  solves this type of issue by channeling the output it receives to an
  arbitrary file descriptor, keeping the writer on the out-of-band
  execution stage.

## Everything is a file {#everything-is-a-file}

The nice thing about the file semantics is that it may solve general
problems for our embedded real-time core:

- it can organize resource management for EVL's kernel objects. If
  every element we export to the user is represented by a file, we can
  leave the hard work of managing the creation and release process to
  the
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
  /dev/evl/*element_type*/. By opening the device file, the
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

- trace channel. We need a way to emit traces from applications
  through the main kernel's FTRACE mechanism for debugging purpose,
  directly from the real-time context. Dovetail readily enables
  tracing from the out-of-band stage, so all we need is an interface
  here.

## EVL device drivers are (almost) common drivers

EVL does not introduce any specific driver model. It exports a
dedicated [kernel API]({{%relref "core/kernel-api/_index.md" %}}) for
implementing real-time I/O operations in common character device
drivers. In fact, the EVL core is composed of a set of such drivers,
implementing each class of elements and utilities.

## Developing the EVL core

Just like Dovetail, developing the EVL core is customary Linux kernel
development, with the addition of dual kernel specific background.

The development tip of the EVL core is maintained in the
_evl/master_ branch of the following GIT repository which tracks
the [mainline
kernel](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git):

  * git://git.evlproject.org/linux-evl.git
  * https://git.evlproject.org/linux-evl.git

This branch is routinely rebased over Dovetail's
[_dovetail/master_]({{% relref
"contents/dovetail/_index.md#developing-dovetail" %}}) branch from the
same repository.
