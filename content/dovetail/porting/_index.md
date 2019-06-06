---
title: "Porting Dovetail"
pre: "&rsaquo; "
weight: 80
---

Porting the Dovetail interface to a different kernel release, or a
different CPU architecture involves enabling the [interrupt
pipeline]({{< relref "dovetail/pipeline/_index.md" >}}) first, then
the [alternate scheduling]({{< relref "dovetail/altsched/_index.md"
>}}) feature which builds on the former.

Interrupt pipelining is enabled by turning on CONFIG_IRQ_PIPELINE:
getting this feature to work flawlessly is a prerequisite before the
rest of the Dovetail port can proceed. At the very least, you should
check that these kernel features get along with interrupt pipelining:

- **CONFIG_PREEMPT**
- **CONFIG_TRACE_IRQFLAGS** (selected by CONFIG_IRQSOFF_TRACER,
  CONFIG_PROVE_LOCKING)
- **CONFIG_LOCKDEP** (selected by CONFIG_PROVE_LOCKING, CONFIG_LOCK_STAT,
  CONFIG_DEBUG_WW_MUTEX_SLOWPATH, CONFIG_DEBUG_LOCK_ALLOC)
- **CONFIG_CPU_IDLE**

Once and only when this layer is rock-solid should you start working
on enabling the alternate scheduling support, which will allow your
autonomous core to schedule tasks created by the kernel. At the end of
this process, turning on CONFIG_DOVETAIL should enable full support
for coupling your autonomous core to the kernel.

To clarify what such porting effort involves, let's have a look at the
main kernel sub-systems/features impacted by the Dovetail-related
changes:

![Alt text](/images/subsystem-port.png "Updated sub-systems")

### IRQFLAGS

This is the most basic change to introduce into the kernel, in order
to turn the architecture-specific API manipulating the CPU's interrupt
flag into the equivalent virtualized calls the interrupt pipeline
provides. By virtualizing these operations, Dovetail keeps the
hardware interrupt events flowing in while still preserving the in-band
kernel code from undue interrupt delivery as explained in [this
document]({{< relref "dovetail/pipeline/_index.md" >}}). You need to
follow [this procedure]({{< relref "dovetail/porting/arch.md" >}}) for
implementing such virtualization.

### ATOMIC OPS

Once the IRQFLAGS have been adapted to interrupt pipelining, the
original atomic operations which rely on explicitly disabling the
hardware interrupts to guarantee atomicity cannot longer work, unless
the call sites are restricted to in-band context, which is not an
option as we will certainly need them for carrying out atomic
operations from out-of-band context too. So we need to iron them in a
way that adds back serialization between callers during updates,
regardless of the caller's context. Both a few generic and
architecture-specific operations need fix ups to achieve this [as
documented here]({{< relref "dovetail/porting/atomic.md" >}}).

### IPI Handling

With SMP-capable hardware, the kernel uses Inter-Processor Interrupts
(aka IPI) to notify remote cores about particular events which may
have happened on the send side. For Dovetail to enable the autonomous
core to trigger and receive some of those special interrupts like any
common (e.g. device) interrupt, some [architecture-specific code is
required]({{< relref "dovetail/porting/arch.md#dealing-with-ipis"
>}}). For instance, a SMP-capable autonomous core will need the
rescheduling IPI Dovetail defines in order to force a remote core to
reschedule its tasks.

### System entry

The system entry starts with low level assembly code receiving
interrupts, traps/exceptions and system calls. For this reason, we
need the following set of changes:

- first, we want to [channel IRQ events to the interrupt pipeline]({{<
  relref "dovetail/porting/arch.md#arch-irq-entry" >}}), instead of
  delivering IRQs directly to the original low-level in-band handler
  (e.g. `handle_arch_irq()` for ARM). With this change in, the
  pipeline can dispatch events immediately to out-of-band handlers if
  any, then conditionally dispatch them to the in-band code too if it
  accepts interrupts, or defer them until it does. This change is a
  prerequisite for enabling the interrupt pipeline.

- because faults/exceptions can happen when running on the out-of-band
  stage (e.g. some task running out-of-band blunders, causing a memory
  access violation), returning from a fault handling must [skip the
  epilogue code]({{< relref "dovetail/porting/arch.md#fault-exit" >}})
  which checks for in-band-specific conditions, such as opportunities
  for userr task rescheduling or/and kernel preemption.

![Alt text](/images/wip.png "To be continued")
