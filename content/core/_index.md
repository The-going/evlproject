---
title: "The EVL core"
menuTitle: "Real-time core"
date: 2019-02-16T16:10:44+01:00
weight: 5
pre: "&#8226; "
---

## Pitching the real-time EVL core

For certain types of applications, offloading a particular set of
time-critical tasks to an autonomous software core embedded into the
Linux kernel may deliver the best performance at the lowest
engineering and runtime costs in comparison to imposing real-time
behavior on the whole kernel logic in order to meet the deadlines
which only those tasks have, like the [native
preemption](https://wiki.linuxfoundation.org/realtime/rtl/blog) model
requires.

In a nutshell, the Xenomai 4 project is about introducing a simple,
scalable and dependable dual kernel architecture for Linux, based on
the [Dovetail interface]({{% relref "dovetail/_index.md" %}}) for
coupling a high-priority software core to the main kernel. This
interface is showcased by a real-time core delivering basic services
to applications via a [straightforward API]({{% relref
"core/user-api/_index.md" %}}). The EVL core is an ongoing development
toward a production-ready real-time infrastructure, which can also be
a starting point for other flavours of dedicated software core
embedded into the Linux kernel. This work is composed of:

- the [Dovetail]({{% relref "dovetail/_index.md" %}}) interface, which
  introduces a high-priority execution stage into the main kernel
  logic, where a functionally-independent software core runs.

- the EVL core which delivers dependable low-latency services to
  applications which have to meet real-time requirements. Applications
  are developed using the common Linux programming model.

- an in-depth documentation which covers both Dovetail and the EVL
  core, with many cross-references between them, so that engineers can
  use the EVL core to support a real-time application, improve it, or
  even implement their own software core of choice on top of Dovetail
  almost by example.

What we are looking for:

- Low engineering and maintenance costs. Working on EVL should only
  require common kernel development knowledge, and the code footprint
  and complexity must remain tractable for small development teams
  (currently about 20 KLOC, which is not even half the size of the
  [Xenomai 3 Cobalt](https://git.xenomai.org/xenomai/-/wikis/home)
  core.

- Low runtime cost. Reliable, ultra low and bounded response time for
  the real-time workload including on low-end, single-core hardware
  with minimum overhead, leaving plenty of CPU cycles for running the
  general purpose workload concurrently.

- High scalability. From single core to high-end multi-core machines
  running real-time workloads in parallel with low and bounded
  latency. Running these workloads on [isolated CPUs]({{< relref
  "core/caveat.md#isolcpus" >}}) significantly improves the
  [worst-case latency figure]({{< relref
  "core/benchmarks/_index.md#stress-load" >}}) in SMP configurations,
  but if your fixture only has one of them, the EVL core should still
  be able to deliver on ultra low and bounded latency.

- Low configuration. We want very few to no runtime tweaks at all to
  be required to ensure the real-time workload is not affected by the
  regular, general purpose workload. Once enabled in the kernel, the
  EVL core should be ready to deliver.

## Make it ordinary, make it simple

The EVL core is a dedicated software core which is embedded into the
kernel, delivering real-time services to applications with stringent
timing requirements. This small core is built like any ordinary
feature of the Linux kernel, not as a foreign extension slapped on top
of it.  [Dovetail]({{% relref "dovetail/_index.md" %}}) plays an
important part here, as it hides the nitty-gritty details of embedding
a companion core into the kernel. Its fairly low code footprint and
limited complexity makes it a good choice as a plug-and-forget
real-time infrastructure, which can also be used as a starting point
for custom core implementations. The following figures have been
obtained from the [CLOC](https://github.com/AlDanial/cloc) tool
counting the lines of source code from the [RTAI](http://rtai.org),
[Xenomai 3 Cobalt](https://git.xenomai.org/xenomai/-/wikis/home) and
[Xenomai 4 EVL](https://evlproject.org/core) core implementation
respectively:

![Alt text](/images/kloc-core.png "EVL kernel code footprint")

The user-space interface to this core is the [EVL library]({{% relref
"core/user-api/_index.md" %}}) (`libevl.so`), which implements the
basic system call wrappers, along with the fundamental thread
synchronization services. No bells and whistles, only the basics. The
intent is to provide simple mechanisms, complex semantics and policies
can and should be implemented in high level APIs based on this library
running in userland.

![Alt text](/images/kloc-user.png "EVL user code footprint")

## Elements {#evl-core-elements}

As the name suggests, _elements_ are the basic features we may require
from the EVL core for supporting real-time applications in this dual
kernel environment. Also, only the kernel could provide such features
in an efficient way, pure user-space code could not deliver. The EVL
core defines six elements:

- [Thread]({{< relref "core/user-api/thread/_index.md" >}}). As the
  basic execution unit, we want it to be runnable either in real-time
  mode or regular GPOS mode alternatively, which exactly maps to
  [Dovetail's]({{% relref "dovetail/pipeline/_index.md" %}}) out-of-band
  and in-band contexts.

- Monitor. This element has the same purpose than the main kernel's
  _futex_, which is about providing an integrated - although much
  simpler - set of fundamental thread synchronization features. Monitors
  are used internally by the EVL library to implement mutexes, [condition
  variables]({{< relref "core/user-api/event/_index.md" >}}), [event
  flag groups]({{< relref "core/user-api/flags/_index.md"
  >}}) and [semaphores]({{< relref "core/user-api/semaphore/_index.md"
  >}}) in user-space.

- [Clock]({{< relref "core/user-api/clock/_index.md" >}}). We may find
  platform-specific clock devices in addition to the core ones defined
  by the architecture, for which ad hoc drivers should be written. The
  clock element ensures all clock drivers present the same interface to
  applications in user-space. In addition, this element can export
  individual software timers to applications which comes in handy for
  running periodic loops or waiting for oneshot events on a specific
  time base.

- [Observable]({{< relref "core/user-api/observable/_index.md"
  >}}). This element is the building block event-driven applications
  can use for implementing the [observer design
  pattern](https://en.wikipedia.org/wiki/Observer_pattern), in which
  any number of observer threads can be notified of updates to any
  number of observable subjects, in a loosely coupled fashion.

- [Cross-buffer]({{< relref "core/user-api/xbuf/_index.md" >}}). A
  cross-buffer (aka _xbuf_) is a bi-directional communication channel
  for exchanging data between out-of-band and in-band thread contexts,
  without impacting the real-time performance on the out-of-band side.
  Any kind of thread (EVL or regular) can wait/poll for input from the
  other side. Cross-buffers serve the same purpose than Xenomai 3's
  _message pipes_ implemented by the _XDDP_ socket protocol.

- [File proxy]({{< relref "core/user-api/proxy/_index.md"
  >}}). Linux-based dual kernel systems are nasty by design: the huge
  set of GPOS features is always visible to applications but they should
  not to use it when they carry out real-time work with the help of the
  autonomous core, or risk unbounded response time. Because of such
  exclusion, issuing I/O file requests such as calling
  [printf(3)](http://man7.org/linux/man-pages/man3/printf.3.html) should
  not be done directly from time-critical loops. A file proxy solves
  such issue by offloading I/O operations on in-band files to dedicated
  workers, keeping the caller on the out-of-band execution stage.

## Everything is a file {#everything-is-a-file}

Each resource exported by EVL to applications is represented by a
file. In addition, each EVL element is associated to a kernel device
object:

- Since all EVL resources are backed by a kernel file internally, the
  hard work of managing their lifetime, preventing stale references by
  tracking their users, is left to the
  [VFS](https://www.kernel.org/doc/Documentation/filesystems/vfs.txt).

- Applications can create [public or private elements]({{< relref
  "core/user-api/_index.md#element-visibility" >}}). A public element
  appears in the EVL device [file hierarchy]({{< relref
  "core/user-api/_index.md#evl-fs-hierarchy" >}}), which enables
  [multi-process applications]({{< relref
  "core/user-api/_index.md#multi-process-apps" >}}) to share elements.

- EVL elements benefit from the permission control, monitoring and
  auditing logic which come with the file semantics.

- [`udev` rules]({{< relref
  "core/user-api/_index.md#device-ownership-and-access" >}}) can be attached to
  events of interest which might happen for any element. Additionally,
  the internal kernel state of elements is exported to user space via
  the `/sys` filesystem.

## EVL device drivers are (almost) common drivers

EVL does not introduce any specific driver model. It exports a
dedicated [kernel API]({{%relref "core/kernel-api/_index.md" %}}) for
implementing real-time I/O operations in common character device
drivers. In fact, the EVL core is composed of a set of such drivers,
implementing each class of elements.

EVL also provides a way to extend existing socket protocol families
with out-of-band I/O capabilities, or add your own protocols via the
new PF_OOB family.

---

{{<lastmodified>}}
