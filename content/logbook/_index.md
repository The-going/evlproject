---
menuTitle: "Logbook"
title: "The Evilian Chronicle"
weight: 30
pre: "&#9656; "
---

### Week 51.2019

Dovetail was ported from kernel 5.4 to kernel 5.5-rc2. Not much fuss,
except maybe on the ARM side with the rebase of the arch-specific vDSO
support (arch/arm/vdso) on the generic vDSO implementation (lib/vdso)
which took place upstream. This turned out to be an opportunity to
rebase the originally ARM-specific [vDSO support for user-mappable
clocksources]({{< relref
"dovetail/porting/clocksource.md#generic-clocksource-vdso" >}}) to the
generic vDSO implementation. Except for these, other changes were
fairly trivial to merge into kernel 5.5-rc2.

Although a much wider test coverage is required, EVL running Dovetail
5.5-rc2 just flies on the handful of boards [tested so far]({{< relref
"ports/_index.md" >}}), including on the [AllWinner
H3](https://linux-sunxi.org/H3) SoC which is now supported.

---

### Week 50.2019

As I was debugging some core [Xenomai](https://xenomai.org/) issue on
x86, I stumbled over the reason for `CONFIG_MAXSMP` breaking the
interrupt pipelines for ages: enabling this kernel feature extends the
range of interrupt vectors to 512K. Guess what would happen with the
3-level interrupt log which cannot address more than 256K vectors both
the I-pipe and Dovetail were using so far? Both gained a 4-level
interrupt log this week to cover the entire vector space when
`CONFIG_MAXSMP` is enabled, which did not go without a vague feeling
of wearing a dunce cap.

---

### Week 49.2019

Added documentation for the EVL [tube]({{< relref
"core/user-api/tube/_index.md" >}}), which is a lightweight and
flexible FIFO data path with a lockless algorithm, and for the [EVL
command]({{< relref "core/commands.md" >}}).

---

### Week 48.2019

The documentation for EVL [mutexes]({{< relref
"core/user-api/mutex/_index.md" >}}), [heaps]({{< relref
"core/user-api/heap/_index.md" >}}) and [polling services]({{< relref
"core/user-api/poll/_index.md" >}}) was significantly updated, along
with details of the [initialization phase]({{< relref
"core/user-api/init/_index.md" >}}) of an application process.

---

### Week 47.2019

EVL [timers]({{< relref "core/user-api/timer/_index.md" >}}) were
documented.

---

### Week 46.2019

EVL [cross-buffers]({{< relref "core/user-api/xbuf/_index.md" >}})
were documented.

---

### Week 44.2019

The week the [EVL core]({{< relref "core/_index.md" >}}) eventually
got rid of the SMP scalability issue which plagues the [Cobalt
core](https://xenomai.org/). The ugly big spinlock (aka _nklock_)
which still serializes all operations inside the former is now gone
from the EVL core, after a long incremental process whick took place
over several months, introducing per-CPU serialization in the core
scheduler, rewriting portions of the EVL thread synchronization
support like waitqueues and mutexes in the same move.

---

### Week 40.2019

Implemented the EVL shim library which mimics the behavior of the EVL
API based on plain POSIX calls from the native *libc. It comes in
handy whenever the real-time guarantees delivered by the EVL core are
not required for quick prototyping or debugging some application
code.

---

### Week 30.2019

Out-of-band IRQs are now flowing faster through the pipeline until the
EVL core can reschedule upon them. We are now on par with the I-pipe
performance-wise, but with a cleaner integration, a less intrusive
implementation, and a much saner locking model for interrupt
descriptors.

---

### Week 29.2019

Since a companion core exhibits a separate scheduler, there is no
point in waiting patiently for the main kernel logic to finish
switching its own tasks before preempting it upon out-of-band
IRQ. Several micro-benchmarks on ARM and arm64 revealed that a
significant portion of the maximum latency was induced by switching
memory contexts atomically. Dovetail now supports generic non-atomic
mm switching for the in-band stage, which allows the EVL core to
preempt in the middle of an in-band memory context switch, reducing
even further the wakeup latency of out-of-band threads under high
memory stress, especially when sluggish outer cache controllers are
part of the picture.

---

### Week 26.2019

The [proxy]({{< relref "core/user-api/proxy/_index.md" >}}) now
supports a 'write granularity' feature, in order to define a fixed
size for writing bulks of data to the target in-band file. Because of
this, the proxy is available for interfacing with in-band files with
special requirements, like writing exactly 64bit words to
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html)
objects, which turns the proxy into yet another mechanism for
[synchronizing out-of-band and in-band threads]({{< relref
"core/user-api/proxy/_index.md#proxy-synchronization-example" >}}).

---

### Week 25.2019

Eventually finalized the out-of-band polling services for EVL.  Since
each instance of an [EVL element]({{< relref
"core/_index.md#evl-core-elements" >}}) is represented by a regular
file, we can wait for events on the corresponding file descriptor via
a common [synchronous multiplexing]({{< relref
"core/user-api/poll/_index.md" >}}) mechanism. For instance, we could
wait for an [event group]({{< relref "core/user-api/event/_index.md"
>}}) to have some event pending, or a [proxy]({{< relref
"core/user-api/proxy/_index.md" >}}) to receive some data from the
in-band peer.

---

### Week 21.2019

The GPIOLIB framework can now handle requests from out-of-band EVL
threads, which means that any GPIO chip driver based on this generic
GPIO core can deliver on ultra-low latency requirements. This is the
first illustration that we need no specific driver model with the EVL
core in order to cope with real-time duties in drivers: we can
leverage [the out-of-band operations]({{< relref
"core/kernel-api/_index.md" >}}) Dovetail provides us as part of the
common file operations defined by the VFS.

---

### Week 19.2019

Just finished porting Dovetail to x86_64, which almost enabled the EVL
core on this architecture in the same move given the very few
arch-specific bits this core requires. Enabling [libevl]({{< relref
"core/user-api/_index.md" >}}) on this architecture was a trivial task
for the same reason. So we now have support for ARM, arm64 and x86_64.

---

### Week 18.2019

Added a scheduling policy for temporal partitioning ([SCHED_TP]({{<
relref "core/user-api/scheduling/_index.md#SCHED_TP" >}})). Useful
when Arinc653-like scheduling is required.

---

### Week 10.2019

Implementation-wise, the support for [EVL timers]({{< relref
"core/user-api/timer/_index.md" >}}) was merged as a sub-function of
the clock element, which simplifies the code.

---

### Week 08.2019

EVL logger and mapper features are now merged into the proxy element.

---

### Week 07.2019

Spent some time writing unit tests for [libevl]({{< relref
"core/user-api/_index.md" >}}). Not exhilarating, but required anyway.

---

### October 8 2018

Thirteen years to the day after I respinned the Xenomai project with
[Xenomai
2](https://mail.rtai.org/pipermail/rtai/2005-October/013222.html), it
seems a good time to articulate the lessons learned from the strengths
and weaknesses of Xenomai in a modern real-time core implementation
based on [Dovetail]({{< relref "dovetail/_index.md" >}}). After months
originally spent working on the 'steely' core, I have decided to send
the whole thing back to the drawing board instead of continuing this
effort: this needs more than a revamping of the implementation, the
way such companion core integrates into the main kernel has to be
re-designed.

This decision eventually led to developing the EVL core, which
originally reused portions of [Cobalt's](https://xenomai.org/) proven
infrastructure such as the scheduler and thread management system. On
the other hand, the kernel-to-core and user-to-core interfaces were
entirely rewritten, dropping RTDM and POSIX entirely. The portion of
inherited bits was expected to decrease over time, especially in the
wake of improving SMP scalability, which eventually happened a year
later.

---

### 2016-2018

The groundwork for Dovetail was laid during this period, by
introducing a new high-priority execution stage into the mainline
kernel logic. On this so-called [out-of-band]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}) stage, we can
run an interrupt pipeline and provide [alternate scheduling]({{<
relref "dovetail/altsched.md" >}}) support to any companion software
core we may want to embed. The main kernel can offload work which has
to meet ultra-low latency requirements to such core, without requiring
the entire kernel machinery to abide by the real-time rules only for a
handful of threads which actually need this.

During this same period, a streamlined version of Xenomai's Cobalt
core codenamed 'steely' which would only provide a POSIX API was
developed for the purpose of testing the Dovetail interface.

---

### Week 50.2015

Started working on Dovetail.

---

{{<lastmodified>}}
