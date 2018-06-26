---
title: "Dovetail interface"
date: 2018-06-26T19:28:38+02:00
weight: 5
pre: "&#9656; "
---

## Introducing Dovetail {#dual-kernel-upsides}

Using Linux as a host for lightweight software cores specialized in
delivering very short and bounded response times has been a popular
way of supporting real-time applications in the embedded space over
the years.

This *dual kernel* design introduces a small real-time infrastructure
into the Linux code, which immediately handles time-critical,
out-of-band activities independently from the ongoing main kernel
work. Application threads co-managed by this infrastructure still
benefit from the common kernel services such as virtual memory
management; they can leverage the rich GPOS feature set Linux provides
such as networking, data storage or GUIs too.

There are significant upsides to keeping the real-time core separate
from the GPOS infrastructure:

- because the two kernels are independent, real-time activities are
  not serialized with GPOS operations internally, removing potential
  delays which might be induced by the non time-critical
  work. Likewise, there is no requirement for keeping the GPOS
  operations fine-grained and highly preemptible at any time, which
  would otherwise induce noticeable overhead on low-end hardware, due
  to the need for pervasive task priority inheritance and IRQ
  threading.

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
defines a new execution stage in the mainline kernel logic.  At any
time, out-of-band activities running on this stage can preempt the
common work. A task-specific software core - such as a real-time core
- can connect to this interface for gaining bounded response time to
external interrupts and ultra-low latency scheduling
capabilities. This translates into the Dovetail implementation as
follows:

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

Dovetail is the successor to the *I-pipe*, the interrupt pipeline
implementation [Xenomai's Cobalt](https://xenomai.org/gitlab/xenomai/)
real-time core currently relies on. The rationale behind this effort
is about securing the maintenance of this key component of dual kernel
systems such as [Xenomai](https://xenomai.org), so that they could be
maintained with common kernel development knowledge, at a fraction of
the engineering and maintenance cost native preemption requires. For
several reasons, the
[I-pipe](https://gitlab.denx.de/Xenomai/xenomai/wikis/Dovetail/) does
not qualify.

Maintaining the I-pipe proved to be difficult over the years as
changes to the mainline kernel regularly caused non-trivial code
conflicts, sometimes nasty regressions to us downstream. Although the
concept of interrupt pipelining proved to be correct in delivering
short response time with reasonably limited changes to the original
kernel, the way this mechanism is currently integrated into the
mainline code shows its age, as explained in [this document]
(https://gitlab.denx.de/Xenomai/xenomai/wikis/Dovetail/).

## Developing Dovetail {#developing-dovetail}

Working on Dovetail is customary Linux kernel development, following
the common set of rules and guidelines which prevails with the
mainline kernel.

The development tip of the Dovetail interface is maintained in the
_dovetail/master_ branch of the following GIT repository which tracks
the [mainline
kernel](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git):

  * git://git.evlproject.org/linux-evl.git
  * https://git.evlproject.org/linux-evl.git

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
