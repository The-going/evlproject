---
menuTitle: "Logbook"
title: "The Evilian Chronicle"
weight: 30
pre: "&#8226; "
---

### Week 18.2020

Bug fixes for the most part, some documentation fixups as well.

We are now tracking kernel v5.7-rc4 and v5.4.36 LTS.

### Week 17.2020

The Observable element was added to the EVL core, as a building block
for implementing the observer design pattern in applications. See
[this document]({{< relref "core/user-api/observable/_index.md" >}})
for more.

### Week 16.2020

At last, the [element visibility attribute]({{< relref
"core/user-api/_index.md#element-visibility" >}}) was introduced. EVL
elements created by the user API can be either publically visible to
other processes, or private to the process which creates them. These
changes induce both an ABI and API bump. Speaking of which, there is
now a manifest of [API revisions]({{< relref
"core/user-api/api-revs.md" >}}) which navigates you to these changes.

On the debugging front, the [SCHED_TP scheduling policy]({{< relref
"core/user-api/scheduling/_index.md#SCHED_TP" >}}) for temporal
partitioning has been fixed and delivers as promised.

Many updates to the online documentation all over the map took place
as well.

We are now tracking kernel v5.7-rc1 and v5.4.32 LTS.

### Week 15.2020

Out-of-band SPI transfers are now available with Dovetail, illustrated
by the [spidev](https://www.kernel.org/doc/Documentation/spi/spidev)
user-space API which now gives access to the feature. Currently, the
BCM2835 SPI and DMA controllers available with the Raspberry PIs are
enabled for out-of-band operations. We are half-way there with NXP's
i.MX series, the SDMA engine is out-of-band capable already but the
eCSPI controller is not there yet. This work goes a long way making
EVL an option for driving closed-control loops with the level of
performance the dual kernel architecture offers, i.e. very low jitter,
high-frequency and low overhead.

We are now tracking kernel v5.4.30 on the LTS branch.

### Week 14.2020

There is an ongoing work on enabling DMA support and SPI transfers
from the out-of-band stage based on cyclic DMA transactions. This work
currently targets DMA and SPI units available with the Raspberry and
i.MX hardware, but the logic is applicable to any flavour. The goal is
to make it possible to implement closed-loop control systems running
at high frequency (10-15 Khz) and/or very low jitter on low-end to
mid-range hardware, e.g. using the regular spidev interface.

We are now tracking kernel v5.6.

### Week 13.2020

The EVL core now supports compat mode, which currently allows running
armv7 binaries on an EVL-enabled armv8 kernel. Typically, if you plan
to run EVL over any of the [Raspberry PI]({{< relref
"core/build-steps.md#enable-kernel-compat-mode" >}}) 64-bit computers,
you are going to like this. Also, the [libevl]({{< relref
"core/user-api/_index.md" >}}) interface headers became C++
friendly. A specific suite of compile-time tests was added, notably to
detect C++ parsing issues early on as `libevl` evolves over time.
Documentation-wise, the [list of ABI revisions]({{< relref
"core/abi-revs.md" >}}) of the EVL core will be maintained from now
on.

We are now tracking kernel v5.6-rc7 and v5.4.27 LTS.

### Week 11-12.2020

Two weeks mostly spent working on the documentation, reorganizing the
GIT tree in the same move to reflect the changes in the [development
and release processes]({{< relref "/devprocess.md" >}}). As a result
of these changes, EVL now tracks the latest LTS release from the
[stable kernel
tree](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git)
(currently v5.4.y), in addition to the development tip of the
[mainline
kernel](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/). A
new section about [getting started]({{< relref "/overview/_index.md"
>}}) with EVL is available.

We are now tracking kernel v5.6-rc6 and v5.4.26 LTS.

### Week 10.2020

A week mostly spent working on the documentation, such as describing
how Dovetail helps in tracking [file descriptors]({{< relref
"/dovetail/files/_index.md" >}}) so that companion cores can implement
their own file-based interface. In addition, more details were added
to the description of the [alternate scheduling]({{< relref
"/dovetail/altsched.md" >}}) interface. Finally, the new [under the
hood]({{< relref "/core/under-the-hood/_index.md" >}}) section was
started, which is intended to help anyone interested in or simply
curious about the "other path to Linux real-time", whether it is
useful for developing your own Linux-based dual kernel system,
contributing to EVL, or educational purpose.

EVL is now tracking kernel v5.6-rc4.

### Week 9.2020

The core and libevl have been made y2038-safe. EVL is now tracking
kernel v5.6-rc3.

The libevl repository shows the new **next** branch, which works in
lockstep with **evl/next** on the kernel side (linux-evl). Once the
latter moves to **evl/master**, **next** should move to **master**.

### Week 8.2020

EVL is now tracking kernel v5.5, heading to v5.6-rc1. A seriously
silly bug in the ARM port was uncovered recently, causing _undefined
instruction_ exceptions to be spuriously reported by the EVL core. If
you run the ARM port of EVL, make sure to pull from the **evl/master**
branch, or at the very least pick [this
commit](https://git.evlproject.org/linux-evl.git/commit/?h=dovetail/master&id=386a332946b72f58793f9a4c72859cb9ff12d4c6)
which is actually reverting a broken and useless change.

Update: EVL was rebased on kernel v5.6-rc2; coming from v5.5.0, this
has been a fairly smooth process. Some upstream changes to the x86
trap handling, and the addition of time namespaces to the generic vDSO
support triggered some issues, but nothing bad or even tricky overall.
This code is available from the new **evl/next** branch,
**evl/master** will stay on v5.5 until v5.6 final is out, as discussed
on the [mailing
list](https://evlproject.org/pipermail/evl/2020-February/000145.html).

### Week 5-7.2020 {#announce-syncbp}

When a common application is being (p)traced by gdb in the so-called
_all-stop mode_, any thread which is stopped (e.g. because it hits a
breakpoint or ^C is pressed) causes all other threads within the same
process to stop as well. As the gdb documentation states, this allows
to examine the overall state of the program, including switching focus
between threads, without worrying that things may change underfoot.

With a companion core sharing the responsibility of scheduling these
threads, enforcing the all-stop mode requires a bit more work than
just sending SIGSTOP to the siblings because the threads currently
running out-of-band may delay stop requests which have to be issued
and handled from the in-band context. In the meantime, the sibling
threads running out-of-band may execute a significant amount of code
before they eventually obey the in-band stop request, which is
precisely what we would like to avoid.

To address this issue, support for synchronous debugger breakpoints is
now available from the EVL core. This feature keeps a thread
(single-)stepped by a debugger synchronized with its siblings from the
same process running in the background, as follows:
    
- as soon as a ptracer (e.g. gdb) regains control over a thread which
  just hit a breakpoint or received SIGINT, sibling threads from the
  same process which run out-of-band are immediately frozen.
    
- all sibling threads which have been frozen are set to wait on a
  common barrier before they can be released. Such release happens
  once all of them have joined the barrier in out-of-band context,
  once the stepped thread resumes. This ensures siblings resume in an
  orderly (i.e. priority-based) manner from a common point in the
  timeline, instead of staggered.

Although the implementation differs, this is inspired from the similar
feature available with [Xenomai 3](https://xenomai.org).

### Week 4.2020

Unboxed the [Raspberry PI
4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b) this
week, and ported EVL to the [Broadcom
BCM2711](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711/README.md)
chip to get a sense of its performance when it comes to real-time
duties. Actually, there was nothing to be ported, at least for the
basic timer and GPIO latency tests, everything just worked out of the
box over EVL v5.5-rc4. For testing EVL with an _armv8_ kernel on the
PI 4, you will need a 64bit root file system too, since EVL does not
support the mixed ABI model yet (i.e. the EVL port to _armv8_ does not
accept EVL system calls issued by executables targeting _armv7_ yet).

As the vendor documentation states, "the architecture of the BCM2711
is a considerable upgrade on that used by the SoCs in earlier Pi
models". Hell yes. It is, at the very least when it comes to real-time
performance. Compared to the PI 3B+ model, worst case latency is
slashed by half for the same tests.

{{% notice warning %}}
Because of past thermal issues with this chip, you really want to
consider [upgrading the
firmware](https://www.tomshardware.com/features/raspberry-pi-4-firmware-cool-temps-network-boot)
for any serious testing with the Raspberry PI 4. Bonus point: you will
get a decent PXE boot in the same move.
{{% /notice %}}

### Week 3.2020

A [new serialization mechanism]({{< relref
"core/kernel-api/stax/_index.md" >}}) was implemented in order to
address a common problem with adding out-of-band support to existing
drivers: how to safely share a portion of the driver logic between the
in-band and out-of-band stages? The idea underlying this mechanism is
based on the observation that dealing with a device involves different
phases: most of which usually have no real-time requirement such as
device set up and channel configuration, only a few may have, such as
exchanging payload data between the application and the device via I/O
transfers.

A typical example would be about accessing an Alsa device: we may want
to control the settings of a device using the mixer application from
the in-band context, or alternatively run a capture/playback loop via
the PCM core from the out-of-band context for achieving bounded
ultra-low latency when exchanging audio samples with the same
device. We may allow the device to be reconfigured when the driver is
not exchanging data with the codec from the out-of-band stage and
conversely, but we do not want both operations to happen concurrently
inside the Alsa core. Because the application layer may not be able to
ensure such operations never overlap (e.g. playing an audio stream
with `aplay` while changing the mixer settings from a different
context using `amixer`), a kernel mechanism which helps in keeping the
general logic safe is welcome.

### Week 2.2020

Documentation about benchmarking EVL is on its way. It will cover the
GPIO latency test, and recommendations for measuring the worst case
latency with EVL and other real-time Linux infrastructures.

### Week 1.2020

The GPIO latency test was merged into the [latmus]({{< relref
"core/testing.md" >}}) program.  This code is paired with a fairly
simple [Zephyr](https://zephyrproject.org)-based application which
implements the latency monitor we need for measuring the response time
of a user-space thread running on the system under test to GPIO events
the monitoring device generates. Using this combo, we can even measure
the response time of non-EVL, strictly single-kernel configurations,
to GPIO events.

Also, Dovetail and EVL were upgraded to kernel 5.5-rc4. Trivial merge,
no issue.

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
contexts atomically. Dovetail now supports generic non-atomic
context switching for the in-band stage which allows the EVL core to
preempt during this process, reducing even further the wakeup latency
of out-of-band threads under high memory stress, especially when
sluggish outer cache controllers are part of the picture.

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
