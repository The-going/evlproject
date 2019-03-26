---
title: "Rules Of Thumb"
date: 2018-06-30T19:02:50+02:00
weight: 20
pre: "&rsaquo; "
---

## Turn on debug options in the kernel configuration!

During the development phase, do yourself a favour: turn on
`CONFIG_DEBUG_IRQ_PIPELINE` and `CONFIG_DEBUG_DOVETAIL`.

The first one will catch many nasty issues, such as calling unsafe
in-band code from out-of-band context. The second one checks the
integrity of the alternate scheduling support, detecting issues in the
architecture port.

The runtime overhead induced by enabling these options is
marginal. Just don't port Dovetail or implement out-of-band client
code without them enabled in your target kernel, seriously.

## Serialize stages when accessing shared data

If some writable data is shared between in-band and out-of-band code,
you have to guard against out-of-band code preempting or racing with
the in-band code which accesses the same data. This is required to
prevent dirty reads and dirty writes:

- one the same CPU, by [disabling interrupts in the CPU]({{% relref
  "dovetail/pipeline/usage/interrupt_protection.md#hard-irq-protection" %}}).

- from different CPUs, by using [hard or mutable spinlocks]({{% relref
  "dovetail/pipeline/usage/locking.md#new-spinlocks" %}}).

## Check that the pipeline torture tests pass

Before any consideration is made to implement out-of-band code on a
platform, check that interrupt pipelining is sane there, by enabling
`CONFIG_IRQ_PIPELINE_TORTURE_TEST` in the configuration. As its name
suggests, this option enables test code which exercizes the [interrupt
pipeline core]({{% relref "dovetail/pipeline/porting/irqflow.md" %}}),
and related features such as the [proxy tick device]({{% relref
"dovetail/pipeline/porting/timer.md" %}}).

{{% notice tip %}}
Since the torture tests need to enable the out-of-band stage for their
own purpose, you may have to disable any Dovetail-based autonomous core in
the kernel configuration for running those tests, like switching off
_CONFIG_EVL_ if the EVL core is enabled.
{{% /notice %}}

When those tests pass, the following output should appear in the
kernel log:
```
Starting IRQ pipeline tests...
IRQ pipeline: high-priority torture stage added.
irq_pipeline-torture: CPU0 initiates stop_machine()
irq_pipeline-torture: CPU3 responds to stop_machine()
irq_pipeline-torture: CPU1 responds to stop_machine()
irq_pipeline-torture: CPU2 responds to stop_machine()
irq_pipeline-torture: CPU1: proxy tick registered (62.50MHz)
irq_pipeline-torture: CPU2: proxy tick registered (62.50MHz)
irq_pipeline-torture: CPU3: proxy tick registered (62.50MHz)
irq_pipeline-torture: CPU0: proxy tick registered (62.50MHz)
irq_pipeline-torture: CPU0: irq_work handled
irq_pipeline-torture: CPU0: in-band->in-band irq_work trigger works
irq_pipeline-torture: CPU0: stage escalation request works
irq_pipeline-torture: CPU0: irq_work handled
irq_pipeline-torture: CPU0: oob->in-band irq_work trigger works
IRQ pipeline: torture stage removed.
IRQ pipeline tests OK.
```

Otherwise, if you observe any issue when running any of those tests,
then the IRQ pipeline definitely needs fixing.

## Know how to differentiate safe from unsafe in-band code {#safe-inband-code}

Not all in-band kernel code is safe to be called from out-of-band
context, actually most of it is **unsafe** for doing so.

A code is deemed safe in this respect when you are 101% sure that it
never does, directly or indirectly, any of the following:

- attempts to reschedule in-band wise, meaning that `schedule()` would
  end up being called. The rule of thumb is that any section of code
  traversing the `might_sleep()` check cannot be called from
  out-of-band context.

- takes a spinlock from any regular type like raw_spinlock_t or
  spinlock_t. The former would affect the virtual interrupt disable
  flag which is invalid outside of the in-band context, the latter
  might reschedule if `CONFIG_PREEMPT` is enabled.

{{% notice tip %}}
In the early days of dual kernel support in Linux, some people would
mistakenly invoke the `do_gettimeofday()` routine from an out-of-band
context in order to get a wallclock timestamp for their real-time
code. Doing so would create a deadlock situation if some in-band code
running `do_gettimeofday()` is preempted by the out-of-band code
re-entering the same routine on the same CPU.  The out-of-band code
would then wait spinning indefinitely for the in-band context to leave the
routine - which won't happen by design - leading to a lockup.  Nowadays,
enabling CONFIG_DEBUG_IRQ_PIPELINE would be enough to detect such mistake
early enough to preserve your mental health.
{{% /notice %}}

## Careful with disabling interrupts in the CPU

When pipelining is enabled, use [hard interrupt protection]({{% relref
"dovetail/pipeline/usage/interrupt_protection.md#hard-irq-protection" %}}) with
caution, especially from in-band code. Not only this might send
latency figures over the top, but this might even cause random lockups
would a rescheduling happen while interrupts are hard disabled.

## Dealing with spinlocks {#spinlock-rule}

Converting regular kernel spinlocks (e.g. `spinlock_t`,
`raw_spin_lock_t`) to hard spinlocks should involve a careful review
of every section covered by such lock. Any such section would then
inherit the following requirements:

- no unsafe in-band kernel service should be called within the
  section.

- the section covered by the lock should be short enough to keep
  interrupt latency low.

## Enable RAW_PRINTK support for printk-like debugging

Unless you are lucky enough to have an ICE for debugging hard issues
involving out-of-band contexts, you might have to resort to basic
printk-style debugging over a serial line. Although the `printk()`
machinery can be used from out-of-band context when Dovetail is
enabled, you should rather use the [`raw_printk()`]({{%relref
"dovetail/pipeline/porting/rawprintk.md" %}}) interface for
this.
