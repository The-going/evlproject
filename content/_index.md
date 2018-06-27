---
title: "Introduction"
date: 2018-06-26T19:28:38+02:00
draft: false
---

# Introduction to Dovetail {#dual-kernel-upsides}

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

Although the real-time infrastructure has to present specific driver
stack and API implementations to applications, there are nonetheless
significant upsides to keeping the real-time core separate from the
GPOS infrastructure:

- because the two kernels are independent, real-time activities are
  not serialized with GPOS operations internally, removing potential
  delays which might be induced by the non time-critical
  work. Likewise, there is no requirement for keeping the GPOS
  operations fine-grained and highly preemptible at any time, which
  would otherwise induce noticeable overhead on low-end hardware, due
  to the requirement for pervasive task priority inheritance and IRQ
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

This documentation presents the two software layers forming Dovetail:
firstly the interrupt pipeline which creates a high-priority execution
stage for a real-time infrastructure, and finally the support for
*shared task control* between the Linux kernel and the real-time
component over the *kthreads* and *user* tasks.

Dovetail is the successor to the *I-pipe*, the interrupt pipeline
implementation [Xenomai's Cobalt](https://xenomai.org/gitlab/xenomai/)
real-time core currently relies on.

{{% notice note %}}
Dovetail only introduces the basic mechanisms for hosting a real-time
core into the Linux kernel, enabling the common programming model for
its applications in user-space. It does **not** implement the
real-time core per se, which should be provided by a separate kernel
component.
{{% /notice %}}
