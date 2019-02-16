---
title: "Prerequisites"
date: 2018-06-27T11:09:03+02:00
---

## Generic requirements

The interrupt pipeline requires the following features to be available
from the target Linux kernel:

- Generic IRQ handling (CONFIG_GENERIC_IRQ), which most architectures
  should support these days.

- IRQ domains (CONFIG_IRQ_DOMAIN).

- Generic clock event abstraction (CONFIG_GENERIC_CLOCKEVENTS).

## Other assumptions

### ARM {#porting-prereq}

- a target ARM machine port must be allowed to specify its own IRQ
  handler at run time (CONFIG_MULTI_IRQ_HANDLER).

- only armv6 CPUs and later are supported, excluding older generations
  of ARM CPUs. Support for ASID (CONFIG_CPU_HAS_ASID) is required.

- machine does not have VIVT cache.

{{% notice warning %}}
armv5 is not supported due to the use of VIVT caches on these
CPUs, which don't cope well - at all - with low latency requirements. [A
work](https://static.lwn.net/images/conf/rtlws11/papers/proc/p01.pdf)
aimed at leveraging the legacy FCSE PID register for reducing the cost
of cache invalidation in context switches has been maintained until
2013 by **Gilles Chanteperdrix**, as part of the legacy *I-pipe* project, Dovetail's ancestor.
This work can still be cloned from this [GIT repository](git://archive.xenomai.org/ipipe-gch.git).
{{% /notice %}}
