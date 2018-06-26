---
menuTitle: "Timer calibration"
title: "Calibrating the core timer"
weight: 2
---

When enabled in the kernel, EVL transparently controls the hardware
timer chip via a [proxy device]({{< relref "dovetail/porting/timer.md"
>}}), serving all timing requests including those originating from the
in-band kernel logic. In order to maximize the timing accuracy, EVL
needs to figure out the basic latency of the target platform.

Upon receipt from an interrupt, the time spent traversing the kernel
code from the low-level entry code until the interrupt handler
installed by some driver is invoked is shorter than the time that
would be required for a kernel thread to resume on such event
instead. It would take even more time for a user-space thread to
resume, since this may entail changing the current memory address
space, which may involve time-consuming MMU-related operations
affecting the CPU caches.

This basic latency may be increased by multiple factors such as:

- bus or CPU cache latency,
- delay required to program the timer chip for the next shot,
- code running with interrupts disabled on the CPU to receive the IRQ,
- inter-processor serialization within the EVL core (_hard spinlocks_).

In order to deliver events as close as possible to the ideal time, EVL
defines the notion of _clock gravity_, which is a static adjustment
value to account for the basic latency of the target system for
responding to timer events from any given clock device, as perceived
by the client code waiting for these wake up events. For this reason,
a clock gravity is defined as a triplet of values, which indicates the
amount of time by which every timer shot should be anticipated from,
depending on the target context it should activate, either IRQ
handler, kernel or user thread.

When started with the _-t_ option, `latmus` runs a series of tests for
determining those best calibration values for the EVL core timer, then
tells the core to use them.

A typical output of this command looks like this:

> Complete core timer calibration
```
# latmus -t
== latmus started for core tuning, period=1000 us (may take a while)
irq gravity...2000 ns
kernel gravity...5000 ns
user gravity...6500 ns
== tuning completed after 34s
```

You might want to restrict the calibration process to specific
context(s), in which case you should pass the corresponding context
modifiers to the `latmus` command, such as _-u_ for user-space and
_-i_ for IRQ latency respectively:

> Context-specific calibration
```
# latmus -tui
== latmus started for core tuning, period=1000 us (may take a while)
irq gravity...1000 ns
user gravity...6000 ns
== tuning completed after 21s
```

{{% notice note %}}
You might get gravity triplets differing from a few microseconds
between runs of the same calibration process: this is normal, and
nothing to worry about provided all those values look close enough to
the expected jitter on the target system. The reason for such
discrepancies is that although _latmus_ does run the same tests time
and again, the conditions on the target system may be different
between runs, leading to slightly varying results (e.g. variations in
CPU cache performance for the calibration loop).
{{% /notice %}}

---

{{<lastmodified>}}
