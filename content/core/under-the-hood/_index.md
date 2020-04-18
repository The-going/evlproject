---
menuTitle: "Under the hood"
title: "Under the hood"
weight: 15
pre: "&#9702; "
---

This (work-in-progress) section will help you navigate the design and
implementation of the [EVL core]({{< relref "core/_index.md" >}}). It
is written from a developer perspective as a practical example of
implementing a companion core living in the Linux kernel, emphasizing
on details about the way the [Dovetail interface]({{< relref
"dovetail/_index.md" >}}) is leveraged for this purpose. This
information is intended to help anyone interested in or simply curious
about the "other path to Linux real-time", whether it is useful for
developing your own Linux-based dual kernel system, contributing to
EVL, or educational purpose.

## How do applications request services from the EVL core? {#hood-syscall-mechanism}

An EVL application interacts with so-called EVL [elements]({{< relref
"core/_index.md#evl-core-elements" >}}). Each element usually exports
a mix of [in-band](http://man7.org/linux/man-pages/man2/ioctl.2.html)
and [out-of-band]({{< relref "core/user-api/io/_index.md" >}}) file
operations implemented by the EVL core (in a few cases, only either
side is implemented). Therefore, an EVL application sees every core
element as a file, and uses a regular file descriptor to interact with
it, either directly or indirectly by calling routines from the
standard [glibc](https://www.gnu.org/software/libc/) or [libevl]({{<
relref "core/user-api/_index.md" >}}) whenever there is a real-time
requirement.

To sum up, an application (indirectly) issues file I/O requests on
element devices to obtain services from the EVL core. The system calls
involved in the interface between an application and the EVL core are
exclusively
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html),
[close(2)](http://man7.org/linux/man-pages/man2/close.2.html),
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html),
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html),
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html),
[mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html) for the
in-band side, and [oob_ioctl()]({{< relref
"core/user-api/io/_index.md#oob_ioctl" >}}), [oob_read()]({{< relref
"core/user-api/io/_index.md#oob_read" >}}), [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) for the out-of-band side.

[libevl]({{< relref "core/user-api/_index.md" >}}) hides the
nitty-gritty details of forming these I/O requests, presenting a
high-level API to be used in applications.

## Where are the EVL core services implemented?

The EVL core can be described as a multi-device driver, each device
type representing an EVL [element]({{< relref
"core/_index.md#evl-core-elements" >}}). From the standpoint of the
Linux device driver model, the EVL core is composed of a set of
character-based device drivers, one per element type. The core is
implemented under the [kernel/evl
hierarchy](https://git.evlproject.org/linux-evl.git/tree/kernel/evl?h=evl/master).

### Element creation {#hood-element-creation}

The basic operation every element driver implements is _cloning_,
which creates a new instance of the element type upon request from the
application. For instance, a [thread element]({{< relref
"core/user-api/thread/_index.md" >}}) is created each time
[evl_attach_self()]({{< relref
"core/user-api/thread/_index.md#evl_attach_self" >}}) is invoked from
[libevl]({{< relref "core/user-api/_index.md" >}}). To do so,
[create_evl_element()](https://git.evlproject.org/libevl.git/tree/lib/internal.c)
sends a request to the special _clone_ device exported by the core at
`/dev/evl/thread/clone`. Upon return, the new thread element is live,
and can be accessed by
[opening](http://man7.org/linux/man-pages/man2/open.2.html) the newly
created device at `/dev/evl/thread/<name>`, where _\<name\>_ is the
thread name passed to [evl_attach_self()]({{< relref
"core/user-api/thread/_index.md#evl_attach_self" >}}). Eventually, the
file descriptor obtained is usable for issuing requests for core
services to that particular element instance via file I/O requests.

The same logic applies to all other types of named elements, such as
[proxies]({{< relref "core/user-api/proxy/_index.md" >}}),
[cross-buffers]({{< relref "core/user-api/xbuf/_index.md" >}}) or
monitors which underlie [mutexes]({{< relref
"core/user-api/mutex/_index.md" >}}), [events]({{< relref
"core/user-api/event/_index.md" >}}), [semaphores]({{< relref
"core/user-api/semaphore/_index.md" >}}) and [flags]({{< relref
"core/user-api/flags/_index.md" >}}).

### Element factory

In order to avoid code duplication, the EVL core implements a
so-called element _factory_. The factory refers to EVL class
descriptors of type [struct
evl_factory](https://git.evlproject.org/linux-evl.git/tree/include/evl/factory.h?h=evl/master),
which describes how a particular element type should be handled by the
generic factory code.

The factory performs the following tasks:

- it [populates the initial device
  hierarchy](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/factory.c?h=evl/master)
  under `/dev/evl` so that applications can issue requests to the EVL
  core. The main aspect of this task is to register a Linux device
  class for each element type, creating the related _clone_ device.

- it implements the generic portion of the
  [ioctl(EVL_IOC_CLONE)](http://man7.org/linux/man-pages/man2/ioctl.2.html)
  request, eventually calling the proper type-specific handler, such
  as
  [thread_factory_build()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/thread.c?h=evl/master)
  for threads,
  [monitor_factory_build()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/monitor.c?h=evl/master)
  for monitors and so on.

- it maintains a reference count on every element instantiated in the
  system, so as to automatically remove elements when they have no
  more referrers. Typically, closing the last file descriptor
  referring to the file underlying an element would cause such removal
  (unless some kernel code is still withholding references to the same
  element).

{{% notice info %}}
Unlike other elements, a thread may exist in absence of any file
reference. The disposal still happens automatically when the thread
exits or voluntarily detaches by calling [evl_detach_self()]({{< relref
"core/user-api/thread/_index.md#evl_attach_self" >}}).
{{% /notice %}}

![Alt text](/images/wip.png "To be continued")
