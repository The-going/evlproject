---
menuTitle: "Clock sources"
title: "Reading clock sources"
weight: 27
---

Your autonomous core is most likely to need a fast access to the
current clock source from out-of-band context, for reading precise
timestamps. The best way to achieve this is by enabling the fast
`clock_gettime()` helper in the [vDSO
support](https://lwn.net/Articles/615809/) for the target CPU
architecture. At least, you will want user-space tasks controlled by
the core to have access to the POSIX-defined `CLOCK_MONOTONIC` and
`CLOCK_REALTIME` clocks from the out-of-band context, with no
execution time penalty due to invoking an [in-band syscall] ({{<
relref "dovetail/altsched/_index.md#inband-switch" >}}).

In most cases, the support for reading the current time base used by
the kernel from the vDSO is already available, and may called from any
task context, including from tasks [running out-of-band]({{< relref
"dovetail/altsched/_index.md#altsched-theory" >}}) without incurring
any [execution stage switch]({{< relref
"dovetail/altsched/_index.md#stage-migration" >}}).

![Alt text](/images/wip.png?height=250px&width=420px "Not there yet")
