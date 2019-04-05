---
title: "Caveat"
weight: 25
pre: "&rsaquo; "
---

## Things you definitely want to know

### **CONFIG_LOCKDEP** is cool but ruins real-time guarantees

The lock dependency engine which helps in tracking down deadlocks and
other locking-related issues is available to Dovetail's [hard
locks]({{% relref "dovetail/pipeline/usage/locking.md#new-spinlocks"
%}}), which underpins most of the serialization mechanisms the EVL
core uses.

This is nice as it has the regular lock validator monitor the hard
spinlocks EVL uses too. However, this comes with a high price
latency-wise: seeing hundreds of microseconds spent in the validator
with hard interrupts off from time to time is not uncommon. Running
the latency monitoring utility (aka `latmus`) which is part of
`libevl` in this configuration should give you pretty ugly numbers.

By enabling any of the following switches in the kernel configuration,
you would implicitly cause CONFIG_LOCKDEP to be enabled too:

- CONFIG_PROVE_LOCKING
- CONFIG_LOCK_STAT
- CONFIG_DEBUG_WW_MUTEX_SLOWPATH
- CONFIG_DEBUG_LOCK_ALLOC

In short, it is fine enabling them for debugging some locking pattern,
but you won't be able to meet real-time requirements at the same time
in such configuration.

### **isolcpus** is our friend too

Isolating some CPUs on the kernel command line using the _isolcpus=_
option, in order to prevent the load balancer from offloading in-band
work to them is not only a good idea with
[PREEMPT_RT](https://wiki.linuxfoundation.org/realtime/rtl/blog), but
for any dual kernel configuration too.

By doing so, having some random in-band work evicting cache lines on a
CPU where real-time threads briefly sleep is less likely, increasing
the odds of costly cache misses, which translates positively into the
latency numbers you can get. Even if EVL's small footprint core has a
limited exposure to such kind of disturbance, saving a handful of
microseconds is worth it when the worst case figure is already within
tenths of microseconds.

### Disable CONFIG_SMP for best latency on single-core systems

On single-core hardware, some out-of-line code may still be executed
for dealing with various types of spinlock with a SMP build, which
translates into additional CPU branches and cache misses. On low end
hardware, this overhead may be noticeable.

Therefore, if you neither need SMP support nor kernel debug options
which depend on instrumenting the spinlock constructs (e.g.
CONFIG_DEBUG_PREEMPT), you may want to disable all the related kernel
options, starting with CONFIG_SMP.
