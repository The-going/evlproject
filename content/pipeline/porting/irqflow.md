---
title: "Interrupt flow"
date: 2018-06-27T15:20:04+02:00
weight: 2
draft: false
---

## Pipelined interrupt flow

Pipelining involves a basic change in controlling the interrupt flow:
`handle_domain_irq()` from the IRQ domain API redirects all parent
IRQs to the pipeline entry by calling `generic_pipeline_irq()`.

> Redirecting the interrupt flow to the pipeline

```markdown
    asm_irq_entry
       -> irqchip_handle_irq()
          -> handle_domain_irq()
             -> generic_pipeline_irq()
                -> irq_flow_handler()
                <IRQ delivery logic>
```

### IRQ flow handlers

Generic flow handlers acknowledge the incoming IRQ event in the
hardware as usual, by calling the appropriate `irqchip` routine
(e.g. `irq_ack()`, `irq_eoi()`). However, the flow handlers do not
immediately invoke the in-band interrupt handlers. Instead, they hand
the event over to the pipeline core by calling `handle_oob_irq()`.

If an out-of-band handler exists for the interrupt received,
`handle_oob_irq()` invokes it immediately, after switching the
execution context to the head stage if not current yet. Otherwise, if
the execution context is currently over the root stage and unstalled,
the pipeline core delivers it immediately to the in-band handler. In
all other cases, the interrupt is deferred, marked as pending into the
current CPU's event log, then the IRQ frame is left.

In absence of out-of-band handler for the event, the device may keep
asserting the interrupt signal until the cause has been lifted in its
own registers. For this reason, the flow handlers as modified by the
pipeline code may have to mask the interrupt line until the in-band
handler has run from the root stage, lifting the interrupt cause. This
typically happens with level-triggered interrupts, preventing the
device from storming the CPU with a continuous interrupt request.

Since all of the IRQ handlers sharing an interrupt line are either
in-band or out-of-band in a mutually exclusive way, such masking
cannot delay out-of-band events.

```markdown
    /* root stage stalled on entry */
    asm_irq_entry
       ...
          -> generic_pipeline_irq()
             ...
                <IRQ logged, delivery deferred>
    asm_irq_exit
    /*
     * CPU allowed to accept interrupts again with IRQ cause not
     * acknowledged in device yet => **IRQ storm**.
     */
    asm_irq_entry
       ...
    asm_irq_exit
    asm_irq_entry
       ...
    asm_irq_exit
```

{{% notice tip %}}
The logic behind masking interrupt lines until events are processed at
some point later - out of the original interrupt context - also
applies to the threaded interrupt model. In this case, interrupt lines
may be masked until the IRQ thread is scheduled in.
{{% /notice %}}

TBD: How to fixup a new flow handler.

## IRQ chip drivers

`irqchip` drivers need to be specifically adapted for supporting the
pipelined interrupt model. The basic task is to ensure that the
following `struct irq_chip` handlers - if defined - can be called from
an out-of-band context safely:

- `irq_mask()`
- `irq_ack()`
- `irq_mask_ack()`
- `irq_eoi()`
- `irq_unmask()`

Such handler is deemed safe to be called from out-of-band context when
it does not invoke **any** in-band kernel service, which might cause
an [invalid context re-entry]({{%relref
"pipeline/optimistic.md#no-inband-reentry" %}}).

The generic IRQ management core serializes calls to `irqchip` handlers
for a given IRQ by serializing access to its interrupt descriptor,
acquiring the per-descriptor `irq_desc::lock` spinlock.  Holding
`irq_desc::lock` when running a handler for any IRQ shared between all
CPUs ensures that a single CPU handles the event.  This - originally -
raw spinlock is automatically turned into a [mutable
spinlock]({{%relref "pipeline/porting/locking.md#new-spinlocks"
%}}) when pipelining interrupts.

In addition, there might be inner spinlocks defined by some `irqchip`
drivers for serializing handlers accessing a common interrupt
controller hardware for _distinct_ IRQs from multiple CPUs
concurrently. Adapting such spinlocked sections found in `irqchip`
drivers to support interrupt pipelining may involve [converting the
related spinlocks]({{%relref "pipeline/rulesofthumb.md#spinlock-rule"
%}}) to hard spinlocks.

Other section of code which were originally serialized by common
interrupt disabling may need to be made fully atomic for running
consistenly in pipelined interrupt mode. This can be done by
introducing hard masking with `hard_local_irq_save()`,
`hard_local_irq_restore()`.

Finally, `IRQCHIP_PIPELINE_SAFE` must be added to `struct
irqchip::flags` member of a pipeline-aware `irqchip` driver, in order
to notify the kernel that such controller can operate in pipelined
interrupt mode.

> Adapting the ARM GIC driver for interrupt pipelining

```
--- a/drivers/irqchip/irq-gic.c
+++ b/drivers/irqchip/irq-gic.c
@@ -93,7 +93,7 @@ struct gic_chip_data {
 
 #ifdef CONFIG_BL_SWITCHER
 
-static DEFINE_RAW_SPINLOCK(cpu_map_lock);
+static DEFINE_HARD_SPINLOCK(cpu_map_lock);
 
 #define gic_lock_irqsave(f)		\
 	raw_spin_lock_irqsave(&cpu_map_lock, (f))
@@ -424,7 +424,8 @@ static const struct irq_chip gic_chip = {
 	.irq_set_irqchip_state	= gic_irq_set_irqchip_state,
 	.flags			= IRQCHIP_SET_TYPE_MASKED |
 				  IRQCHIP_SKIP_SET_WAKE |
-				  IRQCHIP_MASK_ON_SUSPEND,
+				  IRQCHIP_MASK_ON_SUSPEND |
+				  IRQCHIP_PIPELINE_SAFE,
 };
 
 void __init gic_cascade_irq(unsigned int gic_nr, unsigned int irq)
```

{{% notice note %}}
`irq_set_chip()` will complain loudly with a kernel warning whenever
the `irqchip` descriptor passed does not bear the
`IRQCHIP_PIPELINE_SAFE` flag and CONFIG_IRQ_PIPELINE is enabled. Take
this warning as a sure sign that your port of the IRQ pipeline to the
target system is incomplete.
{{% /notice %}}

## Kernel preemption control (PREEMPT)

When pipelining is enabled, `preempt_schedule_irq()` reconciles the
virtual interrupt state - which has not been touched by the assembly
level code upon kernel entry - with basic assumptions made by the
scheduler core, such as entering with (virtual) interrupts disabled.

## Extended IRQ work API {#irq-work}

With interrupt pipelining, due to its NMI-like nature, out-of-band
code running over the head stage might preempt in-band code over the
root stage in the middle of a [critical section]({{%relref
"pipeline/optimistic.md#no-inband-reentry" %}}). For this reason, it
would be unsafe to call any in-band routine from an out-of-band
context.

Triggering in-band work handlers from out-of-band code can be done by
using the regular `irq_work_queue()` service. Such work request from
the head stage is scheduled for running over the root stage on the
issuing CPU as soon as the out-of-band activity quiesces on this
processor. As its name implies, the work handler runs in (in-band)
interrupt context.

{{% notice note %}}
The interrupt pipeline forces the use of a synthetic IRQ as a
notification signal for the IRQ work machinery, instead of a
hardware-specific interrupt vector. This IRQ is labeled "in-band work"
when reported by _/proc/interrupts_.
{{% /notice %}}
