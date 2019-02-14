---
title: "Introduction"
date: 2018-06-26T19:28:38+02:00
draft: false
---

## Introducing Dovetail {#dual-kernel-upsides}

Using Linux as a host for lightweight software cores specialized in
delivering very short and bounded response times has been a popular
way of supporting real-time applications in the embedded space over
the years.

This design - known as the *dual kernel* approach - introduces a small
real-time infrastructure which schedules time-critical activities
independently from the main kernel. Application threads co-managed by
this infrastructure still benefit from the ancillary kernel services
such as virtual memory management, and can also leverage the rich GPOS
feature set Linux provides such as networking, data storage or GUIs.

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

- the functional isolation of the real-time infrastructure from the
  rest of the kernel code restricts common bug hunting to the scope of
  the smaller kernel, excluding most interactions with the very large
  GPOS kernel base.

- with a dedicated infrastructure providing a specific, well-defined
  set of real-time services, applications can unambiguously figure out
  which API calls are available for supporting time-critical work,
  excluding all the rest as being potentially non-deterministic with
  respect to response time.

This documentation presents _Dovetail_, an effort to integrate the
support for interfacing Linux with a real-time component deeply inside
the host kernel's logic.

The two software layers forming Dovetail are described: firstly the
interrupt pipeline which creates a high-priority execution stage for a
real-time infrastructure, and finally the support for [_alternate task
control_]({{% relref "altsched/_index.md" %}}) between the Linux
kernel and the real-time component over the *kthreads* and *user*
tasks.

Although both layers are likely to be needed for implementing a
real-time component, only the interrupt pipeline has to be enabled in
the early stage of porting Dovetail. Support for alternate task
control builds upon the latter, and may - and should - be postponed
until the pipeline is fully functional on the target architecture or
platform. The code base is specifically maintained in a way which
allows such incremental process.

{{% notice note %}}
Dovetail only introduces the basic mechanisms for hosting a real-time
core into the Linux kernel, enabling the common programming model for
its applications in user-space. It does **not** implement the
real-time core per se, which should be provided by a separate kernel
component.
{{% /notice %}}

## Why do we need this?

Dovetail is the successor to the *I-pipe*, the interrupt pipeline
implementation [Xenomai's Cobalt](https://xenomai.org/gitlab/xenomai/)
real-time core currently relies on. The rationale behind this effort
is about securing the maintenance of this key component of dual kernel
systems such as Xenomai, so that they could be maintained with common
kernel development knowledge, at a fraction of the engineering and
maintenance cost native preemption requires. For several reasons, the
I-pipe does not qualify.

Maintaining the I-pipe proved to be difficult over the years as
changes to the mainline kernel regularly caused non-trivial code
conflicts, sometimes nasty regressions to us downstream. Although the
concept of interrupt pipelining proved to be correct in delivering
short response time with reasonably limited changes to the original
kernel, the way this mechanism is currently integrated into the
mainline code shows its age, as explained in [this document]
(https://gitlab.denx.de/Xenomai/xenomai/wikis/Dovetail/).

## Audience

The intended audience of this document is people having common kernel
development knowledge, who may be interested in building a real-time
component on Dovetail, porting it to their architecture or platform of
choice. Knowing about the basics of interrupt flow, IRQ chip and clock
event device drivers in the kernel tree would be a requirement for
porting this code.

However, this document is not suited for getting one's feet wet with
kernel development, as it assumes that the reader is already familiar
with kernel fundamentals.
