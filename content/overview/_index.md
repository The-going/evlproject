---
title: "Overview"
menuTitle: "Get Started"
weight: 1
pre: "&#8226; "
---

{{% mixedgrid src="/images/overview-design.png" %}}
**A dual kernel architecture**. EVL brings real-time capabilities to
Linux by embedding a companion core into the kernel, which
specifically deals with tasks requiring ultra low and bounded response
time to events. For this reason, this approach is known as a _dual
kernel architecture_, delivering stringent real-time guarantees to
some tasks alongside rich operating system services to others. In this
model, the general purpose kernel and the real-time core operate
almost asynchronously, both serving their own set of tasks, always
giving the latter precedence over the former.
{{% /mixedgrid %}}

In order to achieve this, the EVL project works on three components:

- a piece of inner kernel code - aka [Dovetail]({{< relref
"dovetail/_index.md" >}}) - which acts as an interface between the
companion core and the general purpose kernel. This layer introduces a
[high-priority execution stage]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}) on which tasks
with real-time requirements should run. In other words, with the
Dovetail code in, the Linux kernel can plan for running out-of-band
tasks in a separate execution context which is not subject to the
common forms of serialization the general purpose work has to abide by
(e.g. interrupt masking, spinlocks).

- a compact and scalable [real-time core]({{< relref "core/_index.md"
>}}), which is intended to serve as a reference implementation for
other dual kernel systems based on Dovetail.

- a library known as [libevl]({{< relref "core/user-api/_index.md"
>}}), which enables invoking the real-time core services from
applications.

Originally, Dovetail [forked
off](https://xenomai.org/pipermail/xenomai/2018-June/039002.html) of
the [I-pipe](https://gitlab.denx.de/Xenomai/xenomai/wikis/home)
interface driven by the [Xenomai project](https://xenomai.org/),
mainly to address a fundamental maintenance issue with respect to
tracking the most recent kernel releases. In parallel, a simplified
variant of the [Cobalt
core](https://xenomai.org/documentation/xenomai-3/html/xeno3prm/group__cobalt__core.html)
once known as 'Steely' was implemented in order to experiment freely
with Dovetail as the latter significantly departed from the I-pipe
interface. At some point, Steely evolved into an even simpler, basic
and more scalable real-time core which now serves as the reference
implementation for Dovetail, known as the [EVL core]({{< relref
"/core/_index.md" >}}).
  
### When is a dual kernel architecture a good fit?

A dual kernel system should keep the functional overlap between the
general purpose kernel and the companion core minimal, only to hand
over the time-critical workload to a dedicated component which is
simple and decoupled enough from the rest of the system for you to
trust. Typical applications best-served by such infrastructure have to
acquire data from external devices with only small jitter within a few
tenths of microseconds (absolute [worst case]({{< relref
"core/benchmarks/_index.md#stress-load" >}})) once available, process
such input over POSIX threads which can meet real-time requirements by
running on the [high-priority stage]({{< relref "dovetail/altsched.md"
>}}), offloading any non-(time-)critical work to common
threads. Generally speaking, such approach like the EVL core
implements may be a good fit for the job in the following cases:

- if your application needs ultra-low response times and/or strictly
  limited jitter in a reliable fashion. Reliable as in &laquo; not
  impacted by any valid kernel or user code the general purpose kernel
  might run in parallel in a way which could prevent stringent
  real-time deadlines from being met &raquo;. Valid code in this case
  meaning not causing machine crashes; the companion core of a dual
  kernel system is not sensitive to slowdowns which might be induced
  by poorly written general purpose drivers. For instance, a low
  priority workload can put a strain on the CPU cache subsystem,
  causing delays for the real-time activities when it resumes for
  handling some external event:
  
  - if this workload manipulates a large data set continuously,
    causing frequent cache evictions. As the outer cache in the
    hierarchy is shared between CPUs, a ripple effect does exist on
    all of them, including the [isolated
    ones](https://elinux.org/CPU_Shielding_capability).

  - if this workload involves many concurrent threads causing a high
    rate of context switches, which may get even worse if those
    threads belong to different processes (i.e. having distinct
    address spaces).

  	The small footprint of the dedicated core helps in this case,
  since less code and data are involved in managing the real-time
  system as a whole, lowering the impact of unfavorable cache
  conditions. In addition, the small core does not have to abide by
  the locking rules imposed on the general purpose kernel code when
  scheduling its threads. Instead, it may preempt it at any time,
  based on the [interrupt pipelining]({{< relref
  "dovetail/pipeline/_index.md" >}}) technique Dovetail
  implements. This [piece of information]({{ relref
  "core/benchmarks/_index.md#stress-load" }}) describes typical
  runtime situations when the general purpose workload is putting
  pressure on the overall system, regardless of the relative priority
  of its tasks.

- if your application system design _requires_ the real-time execution
  path to be logically isolated from the general purpose activities by
  construction, so as not to share critical subsystems like the common
  scheduler.

- if resorting to [CPU
  isolation](https://elinux.org/CPU_Shielding_capability) in order to
  mitigate the adverse effect the non real-time workload might have on
  the real-time side is not an option, or once tested, is not good
  enough for your use case. Obviously, using such trick with low-end
  hardware on a single-core CPU would not fly since at least one
  non-isolated CPU must be available to the kernel for carrying out
  the system housekeeping duties.

### What does a dual kernel architecture require from your application? {#dual-kernel-requirements}

In order to meet the deadlines, a dual kernel architecture requires
your application to exclusively use the dedicated system call
interface the real-time core implements. With the EVL core, this API
is [libevl]({{< relref "core/user-api/_index.md" >}}), or any other
high-level API based on it which abides by this rule. Any regular
Linux system call issued while running on the high-priority stage
would automatically [demote the caller]({{< relref
"dovetail/altsched.md#inband-switch" >}}) to the low priority stage,
dropping real-time guarantees in the process.

> This rule has consequences when using C++ for instance, which
> requires to define the set of usable classes and runtime features
> which may be available to the application.

This said, your application may use any service from your favorite
standard C/C++ library outside of the time-critical context.

Generally speaking, a clear separation of the real-time workload from
the rest of the implementation in your application is key. Having such
a split in place should be a rule of thumb regardless of the type of
real-time approach including with native preemption, it is **crucial**
with a dual kernel architecture. Besides, this calls for a clear
definition of the (few) interface(s) which may exist between the
real-time and general purpose tasks, which is definitely the right
thing to do.

The basic execution unit the EVL core recognizes for delivering its
services in user space is the [thread]({{< relref
"core/user-api/thread/_index.md" >}}), which commonly translates as
[POSIX
threads](http://man7.org/linux/man-pages/man3/pthread_create.3.html)
in user space.

### Which API should be used for implementing real-time device drivers?

The EVL core exports a [kernel API]({{< relref
"core/kernel-api/_index.md" >}}) for writing drivers, extending the
Linux device driver interface so that applications can benefit from
out-of-band services delivered from the high-priority execution stage.

The basic execution unit the EVL core recognizes for delivering its
services in kernel space is the [kthread]({{< relref
"core/kernel-api/kthread/_index.md" >}}), which is a common Linux
kthread on EVL steroids.

### What is the dual kernel code footprint?

All figures reported in the charts below below have been determined by
[CLOC](https://github.com/AlDanial/cloc), only retaining C and
assembly source files in the comparisons.

{{% mixedgrid-small src="/images/cloc-dovetail.png" %}}
> **Dovetail footprint on kernel code.** The code footprint of the
Dovetail interface is 7.8 Kloc added to the kernel as of v5.7-rc5.
Most changes happen in the generic kernel and driver code, which amount
for 73% of the total. The rest is split into ARM, arm64 and x86-specific code.
An architecture-specific port represents less than 10% of the total on
average.
{{% /mixedgrid-small %}}

{{% mixedgrid-small src="/images/cloc-evl.png" %}}
> **EVL core footprint on kernel code.** The EVL core on top of
Dovetail is 15.2 Kloc. 97% of this code is architecture-agnostic. Each
architecture port amounts for 1% of the rest, which is 163 lines of
code on average. This shows that Dovetail is actually responsible for
the overwhelming majority of the architecture-specific support a
companion core should need.
{{% /mixedgrid-small %}}

{{% mixedgrid-small src="/images/cloc-evl-in-kernel.png" %}}
> **Overall dual kernel code footprint.** The overall footprint of
the EVL dual kernel system amounts to 26 Kloc, which includes Dovetail,
the EVL core and its out-of-band capable drivers so far. This is 0.13% of
the total kernel code base as of v5.7-rc5.
{{% /mixedgrid-small %}}

{{% mixedgrid-small src="/images/cloc-ipipe.png" %}}
> **Comparing I-pipe and Dovetail footprints.** These figures compare
the latest [I-pipe implementation](https://gitlab.denx.de/Xenomai/xenomai/wikis/home)
available to date based on kernel
v4.19.x with Dovetail for v5.7-rc5. Dovetail provides additional core
services such as built-in out-of-band task scheduling support which,
as a consequence companion cores don't have to implement for each CPU
architecture. The ARM-specific code is notably smaller
for Dovetail, thanks to a better integration of the
[interrupt pipeline]({{< relref "dovetail/pipeline/_index.md" >}}) logic
within the mainline kernel.
{{% /mixedgrid-small %}}

{{% mixedgrid-small src="/images/cloc-evl-vs-xenomai.png" %}} >
> **Comparing Cobalt and the EVL core footprints.** These figures
compare Xenomai 3.1 with EVL for kernel v5.7-rc5. The drastic
reduction of the code footprint EVL shows is mainly due to focusing on
a simpler yet flexible [feature set]({{< relref "core/_index.md#evl-core-elements" >}})
and reusing the [common driver
model]({{< relref "core/oob-drivers/_index.md" >}}).  Besides, most of
the architecture-specific code is handled by Dovetail, unlike Cobalt
which still has to deal with the nitty-gritty details of task
switching, like FPU management.
{{% /mixedgrid-small %}}

### Porting Dovetail

If you intend to port Dovetail to:

   - some arbitrary kernel flavor or release.

   - an unsupported hardware platform.

   - another CPU architecture.

Then you could make good use of the following information:

- first of all, the EVL development process is described in this
  [document]({{< relref "devprocess.md" >}}). You will need this
  information in order to track the EVL development your own work is
  based on.

- detailed information about porting the Dovetail interface to another
  CPU architecture for the Linux kernel [is given here]({{< relref
  "dovetail/porting/_index.md" >}}).

- the current collection of &laquo; rules of thumb &raquo; when it
  comes to developing software on top of EVL's dual kernel
  infrastructure.

### Implementing your own companion core

If you plan to develop your own core to embed into the Linux kernel
for running POSIX threads on the [high-priority stage]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}) Dovetail
introduces, you can use the EVL core implementation as a reference
code with respect to interfacing your work with the general purpose
kernel. To help you further in this task, you can refer to the
following sections of this site:

- all documentation sections mentioned earlier about porting Dovetail.

- a description of the so-called [alternate scheduling]({{< relref
  "dovetail/altsched.md" >}}) scheme, by which Linux kernel threads
  and POSIX threads may gain access to the high-prority execution
  stage in order to benefit from the real-time scheduling guarantees.

- a developing series of [technical documents]({{< relref
  "core/under-the-hood/_index.md" >}}) which navigates you through the
  EVL core implementation.

### Running the EVL core

#### Recipe for the impatient {#evl-recipe-for-impatient}

1. read this [document]({{< relref "core/build-steps.md" >}}) about
building the EVL core and libevl.

2. boot your EVL-enabled kernel.

3. write your first application code using the [libevl API]({{< relref
"core/user-api/_index.md" >}}). You may find the following bits
useful, particularly when discovering the system:

       - what does [initializing an EVL application]({{< relref
         "core/user-api/init/_index.md" >}}) entail.

       - how to have [POSIX threads run on the high-priority
       stage]({{< relref "core/user-api/thread/_index.md" >}}).

       - which is the proper [calling context]({{< relref
       "core/user-api/function_index/_index.md" >}}) for each EVL
       service from this API.

4. [calibrate]({{< relref "core/runtime-settings.md" >}}) and
   [test]({{< relref "core/testing.md" >}}) the system.

#### For the rest of us

The process for getting the EVL core running on your target system can
be summarized as follows (click on the steps to open the related
documentation):

{{<mermaid align="left">}}
graph LR;
    S("Build libevl") --> X["Install libevl"]
    style S fill:#99ccff;
    click S "/core/build-steps#building-libevl"
    X --> A["Build kernel"]
    click A "/core/build-steps#building-evl-core"
    A --> B["Install kernel"]
    style A fill:#99ccff;
    B --> C["Boot target"]
    style C fill:#ffffcc;
    C --> D["Run unit tests"]
    style D fill:#ffffcc;
    click D "/core/testing#evl-unit-testing"
    D --> L{OK?}
    style L fill:#fff;
    L -->|Yes| E["Test with 'hectic'"]
    L -->|No| Z["Report upstream"]
    style Z fill:#ff420e;
    click E "/core/testing#hectic-program"
    click Z "https://evlproject.org/mailman/listinfo/evl/"
    E --> M{OK?}
    style M fill:#fff;
    M -->|Yes| F["Calibrate timer"]
    M -->|No| Z
    click F "/core/runtime-settings#calibrate-core-timer"
    style E fill:#ffffcc;
    F --> G["Test with 'latmus'"]
    style F fill:#ffffcc;
    style G fill:#ffffcc;
    click G "/core/testing#latmus-program"
    G --> N{OK?}
    style N fill:#fff;
    N -->|Yes| O["Go celebrate"]
    N -->|No| R{Kconfig issue?}
    style O fill:#33cc66;
    style R fill:#fff;
    click R "/core/caveat"
    R -->|Yes| A
    R -->|No| Z
{{< /mermaid >}}

Once the EVL core runs on your target system, you can go directly to
step #3 of the [quick recipe]({{< relref "#evl-recipe-for-impatient"
>}}) above.
