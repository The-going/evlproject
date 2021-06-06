---
title: "Dovetail interface"
date: 2018-06-26T19:28:38+02:00
weight: 10
pre: "&#8226; "
---

## Introducing Dovetail {#dual-kernel-upsides}

Using Linux as a host for lightweight software cores specialized in
delivering very short and bounded response times has been a popular
way of supporting real-time applications in the embedded space over
the years.

This *dual kernel* design introduces a small real-time infrastructure
into the Linux kernel, which immediately handles time-critical,
out-of-band activities independently from the ongoing general purpose
kernel work. Application threads co-managed by this infrastructure
still benefit from the common kernel services such as virtual memory
management; they can leverage the rich GPOS feature set Linux provides
such as networking, data storage or GUIs too.

There are significant upsides to keeping the real-time core separate
from the GPOS infrastructure:

- because the autonomous core does not depend on the main kernel logic
  for running time-critical operations, real-time activities are not
  serialized with GPOS operations internally. This removes potential
  delays induced by the latter, by construction. As a result, there is
  no need for keeping all GPOS operations fine-grained and highly
  preemptible at any time, which may otherwise induce noticeable
  overhead as task priority inheritance and IRQ threading have to
  apply system-wide, affecting all tasks including those which have
  zero real-time requirement. To sum up, the lesser the effort for the
  kernel to maintain real-time guarantees internally, the more CPU
  bandwidth is available to applications.

- when debugging a real-time issue, the functional isolation of the
  real-time infrastructure from the rest of the kernel code restricts
  bug hunting to the scope of the small autonomous core, excluding
  most interactions with the very large GPOS kernel base.

- with a dedicated infrastructure providing a specific, well-defined
  set of real-time services, applications can unambiguously figure out
  which API calls are available for supporting time-critical work,
  excluding all the rest as being potentially non-deterministic with
  respect to response time. \
  \
  Said differently, would you assume that each and every routine from
  the _glibc_ becomes real-time capable solely by virtue of running on a
  native preemption system? Of course you would not, therefore you would
  carefully select the set of services your real-time application may
  call from its time-critical work loop in any case. For this reason,
  providing a compact, dedicated API which exports a set of services
  specifically aimed at real-time usage is clearly an asset, not a
  limitation.

This documentation presents _Dovetail_, a kernel interface which
introduces a high-priority execution stage into the main kernel logic.
At any time, out-of-band activities running on this stage can preempt
the common work. A task-specific software core - such as a [real-time
core]({{< relref "core/_index.md" >}}) - can connect to this interface
for gaining bounded response time to external interrupts and ultra-low
latency scheduling capabilities. This translates into the Dovetail
implementation as follows:

- the interrupt pipeline which creates a high-priority execution stage
for an autonomous software core to run on.

- support for the so-called [_alternate scheduling_]({{% relref
"dovetail/altsched.md" %}}) between the main kernel and the autonomous
software core for sharing *kthreads* and *user* tasks.

Although both layers are likely to be needed for implementing some
autonomous core, only the interrupt pipeline has to be enabled in the
early stage of porting Dovetail. Support for alternate scheduling
builds upon the latter, and may - and should - be postponed until the
pipeline is fully functional on the target architecture or
platform. The code base is specifically maintained in a way which
allows such incremental process.

{{% notice note %}}
Dovetail only introduces the basic mechanisms for hosting an
autonomous core into the kernel, enabling the common programming model
for its applications in user-space. It does **not** implement the
software core per se, which should be provided by a separate kernel
component instead, such as the [EVL core]({{% relref "core/_index.md" %}}).
{{% /notice %}}

## Why do we need this?

Linux-based dual kernel systems require some interface layer to couple
the secondary core (typically a real-time capable one like the [EVL
core]({{< relref "core/_index.md" >}})) to the logic of the kernel it
is embedded in, so as to benefit from the rich Linux feature set while
running dedicated applications with stringent real-time
requirements. The archetypical implementation of such kind of
interface is the [I-pipe](https://git.xenomai.org/xenomai/wikis/home),
which served both [RTAI](http://rtai.org) and [Xenomai 3
Cobalt](https://git.xenomai.org/xenomai/wikis/home) over the
years. For several reasons explained [in this
document](https://git.xenomai.org/xenomai/wikis/Dovetail),
maintaining the I-pipe proved to be difficult as changes to the
mainline kernel regularly caused non-trivial code conflicts, sometimes
nasty regressions to the I-pipe maintainers downstream. Although the
concept of interrupt pipelining proved to be correct in delivering
short response time with reasonably limited changes to the original
kernel, the way this mechanism is integrated into the mainline code
shows its age.

Dovetail is the successor to the **I-pipe**, with the following goals:

- introduce a [high-priority execution stage]({{< relref
  "dovetail/pipeline/_index.md#two-stage-pipeline" >}}) for
  time-critical operations into the kernel logic, enabling all device
  interrupts to behave like NMIs from the standpoint of the kernel.

- provide a straightforward interface to autonomous cores for running
  Linux tasks on this high-priority execution stage when they have to,
  enabling ultra-fine grained preemption of all other kernel
  activities in such an event.

- enable the common Linux programming model for applications which
  should be controlled by the autonomous core for delivering ultra-low
  latency services (private user address space, multi-threading, SMP
  capabilities, system calls etc).

- make it possible to maintain Dovetail (and ultimately the [EVL
  core]({{< relref "core/_index.md" >}}) which is based on it) with
  common kernel development knowledge, at a fraction of the
  engineering and maintenance cost native preemption requires.
  Typically, Dovetail always favors extending existing kernel
  subsystems with the ability to deal with the new execution stage,
  instead of taking sideways steps. For instance, the [interrupt
  pipeline logic]({{< relref "dovetail/pipeline/irq_handling.md" >}})
  is directly integrated into the [generic IRQ
  layer](https://www.kernel.org/doc/html/latest/core-api/genericirq.html).

## Developing Dovetail {#developing-dovetail}

Working on Dovetail is customary Linux kernel development, following
the common set of rules and guidelines which prevails with the
mainline kernel.

The Dovetail interface is maintained in the following GIT repository
which tracks the [mainline
kernel](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git):

  * git@git.xenomai.org:Xenomai/linux-dovetail.git
  * https://git.xenomai.org/linux-dovetail.git

## Audience

This document is intended to people having common kernel development
knowledge, who may be interested in building an autonomous software
core on Dovetail, porting it to their architecture or platform of
choice. Knowing about the basics of interrupt flow, IRQ chip and clock
event device drivers in the kernel tree would be a requirement for
porting this code.

However, this document is not suited for getting one's feet wet with
kernel development, as it assumes that the reader is already familiar
with kernel fundamentals.

---

{{<lastmodified>}}
