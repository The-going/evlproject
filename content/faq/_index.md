---
title: "FAQ"
weight: 25
pre: "&#9656; "
draft: true
---

### Why another dual kernel system?

Why a dual kernel system based on Linux is still relevant when
[real-time
preemption](https://wiki.linuxfoundation.org/realtime/rtl/blog)
capabilities will soon be available from the stock mainline kernel is
addressed by [this document]({{%relref
"dovetail/_index.md#dual-kernel-upsides" %}}). Why the [EVL
core]({{%relref "core/_index.md" %}}) can better illustrate the deeper
integration between a dedicated real-time core and the GPOS kernel
than [Xenomai](https://xenomai.org/) - which is arguably one of the
most deployed dual kernel systems to date - is explained next.

Xenomai's _Cobalt_ real-time core and the interrupt pipeline support
(aka _I-pipe_) it depends on have limitations which could not be
lifted without changing fundamental aspects of their respective
implementation and interfaces:

- a tendency for many Xenomai drivers to duplicate mainline code and
  eventually bit rot. Such a driver which originates from the mainline
  kernel is significantly modified in order to convert it to
  [RTDM](https://xenomai.org/documentation/xenomai-3/html/xeno3prm/group__rtdm.html),
  Xenomai's own device driver model. Usually, the code from the
  original driver which is not deemed useful for carrying out
  real-time work is stripped out from the Xenomai copy, and the
  resulting software has to live out-of-tree (kernel-wise). For these
  reasons, upstream changes to the original driver are rarely
  backported to its Xenomai counterpart because this is a tedious and
  error-prone task done manually, which is an obvious maintenance
  problem. This issue is particularly significant with all NIC drivers
  available from Xenomai's real-time networking stack (_RTnet_).

- a single lock model (aka _nklock_) in _Cobalt_ which serializes all
  CPU cores running real-time code inside most critical sections. This
  straightforward locking model has benefited Xenomai for years thanks
  to its simplicity and reliability with only a few CPU cores, but it
  shows its age in a time when 8+ cores per CPUs are frequently found,
  including in the embedded space.

- a code footprint which is still too large and complex in kernel
  space despite a major reduction with the Xenomai 3.x series, due to
  a POSIX interface directly implemented there which amounts to 100+
  system calls. Among other issues, this makes the barrier to entry
  for new contributors uselessly high.

- the interrupt management, IRQ handling, clock source management are
  all implemented mainly by redundant code in the _I-pipe_ which runs
  in parallel to the mainline kernel's logic. Although efforts were
  made over the years to simplify and better integrate the interrupt
  pipelining mechanism into the mainline code, some design decisions
  for the _I-pipe_ block further improvements in this area. As a
  consequence, limitations are imposed on Xenomai, such as the
  inability to deal reliably with power management events or even
  survive dynamic frequency scaling in CPUs, which is kind of a
  problem (although CPU throttling is [generally bad news for latency
  figures]({{% relref "core/caveat/_index.md#caveat-cpufreq" %}}), but
  the real-time core should not choke on it at the very least).

Facing that, two options were possible: either fixing those
fundamental issues gradually in the _Cobalt_ core over time, at the
expense of breaking compatibility for many RTDM-based drivers already
in the field (e.g. _nklock_ removal), supporting even more kernel
interface wrappers (e.g. some RTDM-like shim layer), and eventually
implementing a new Xenomai POSIX interface from scratch in user-space
based on a new set of basic kernel services. Or, starting with a clean
sheet, for the purpose of implementing a reference dual kernel system
which could be straightforward and efficient enough for others to use
directly or as a practical example for developing their own system,
which is the idea behind the EVL project. This has led to the EVL
real-time core, which addresses the issues presented earlier as
follows:

- the EVL core does not introduce a specific driver model, but
  marginally extends the regular Linux model for supporting
  [out-of-band I/O operations]({{% relref "core/kernel-api/_index.md"
  %}}) from the real-time core instead.

- it is scalable on large multi-core systems, there is **no big
  lock**.

- it implements a handful of basic features in kernel space known as
  [_elements_]({{% relref "core/_index.md#evl-core-elements" %}}),
  which are building blocks for high-level API services in user-space.

- it is [Dovetail]({{% relref "dovetail/_index.md" %}})-native and was
  actually used as a test bench for this new dual kernel interface
  which solves the existing design issues of the _I-pipe_.

### Who is behind the EVL project?

The guy who started [this stuff](https://xenomai.org) nearly two
decades ago, then [that one](https://lwn.net/Articles/1743/) too,
which eventually became
[this](https://lwn.net/Articles/140374/). Hopefully, at some point, I
will get the dual kernel thing right eventually.

### Where should the discussion on EVL take place?

You can register on the [EVL mailing
list](https://evlproject.org/mailman/listinfo/evl/) to discuss
EVL-related topics including Dovetail and the real-time EVL core.
