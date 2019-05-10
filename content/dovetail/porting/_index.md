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

![Alt text](/images/wip.png "To be continued")
