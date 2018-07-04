---
title: "Interrupt flow"
date: 2018-06-27T15:20:04+02:00
weight: 2
draft: false
---

## Pipelined interrupt flow

Interrupt pipelining involves a basic change in controlling the
interrupt flow: `handle_domain_irq()` from the IRQ domain API
redirects all parent IRQs to the pipeline entry by calling
`generic_pipeline_irq()`.

> Redirecting the interrupt flow to the pipeline

```markdown
    asm_irq_entry
       -> irqchip_handle_irq()
          -> handle_domain_irq()
             -> generic_pipeline_irq()
                -> irq_flow_handler()
                   -> handle_oob_irq()
```

### IRQ flow handling

Generic flow handlers acknowledge the incoming IRQ event in the
hardware as usual, by calling the appropriate `irqchip` routine
(e.g. `irq_ack()`, `irq_eoi()`) according to the interrupt
type. However, the flow handlers do not immediately invoke the in-band
interrupt handlers. Instead, they hand the event over to the pipeline
core by calling `handle_oob_irq()`.

If an out-of-band handler exists for the interrupt received,
`handle_oob_irq()` invokes it immediately, after switching the
execution context to the head stage if not current yet. Otherwise, the
event is marked as pending in the root stage's log for the current
CPU.

{{% notice tip %}}
This is important to notice: _every_ interrupt which is _not_ handled
by an out-of-band handler will end up into the root stage's event
log. This means that all external interrupts must have a handler in
the in-band code - which should be the case for a sane kernel anyway.
{{% /notice %}}

Once `generic_pipeline_irq()` has returned, if the preempted execution
context was running over the root stage unstalled, the pipeline core
synchronizes the interrupt state immediately, meaning that all IRQs
found pending in the root stage's log are immediately delivered to
their respective in-band handlers. In all other situations, the IRQ
frame is left immediately without running those handlers. The IRQs may
remain pending until the in-band code resumes from preemption, then
clears the [virtual interrupt disable flag]({{%relref
"pipeline/optimistic.md#virtual-i-flag" %}}), which would cause the
interrupt state to be synchronized, running the in-band handlers
eventually.

For delivering an IRQ to the in-band handlers, the interrupt flow
handler is called again by the pipeline core. When this happens, the
flow handler processes the interrupt as usual, skipping the call to
`handle_oob_irq()` though.

{{% notice tip %}}
Yes, you did read it right: interrupt flow handlers may run twice for
a single IRQ in Dovetail's pipelined interrupt model: firstly to
submit the event immediately to any out-of-band handler which may be
interested in it, finally to run the in-band handler(s) accepting that
event if its was not delivered to any out-of-band handler. You may
construe the meaning of calling `handle_oob_irq()` as _"let's poll for
any out-of-band handler which might be interested in this event"_.
This also means that any interrupt can have either an in-band or an
out-of-band handler, but not both.
{{% /notice %}}

In absence of any out-of-band handler for the event, the device may
keep asserting the interrupt signal until the cause has been lifted in
its own registers. At the same time, we might not be allowed to run
the in-band handler immediately over the current interrupt context if
the root stage is currently stalled, we would have to wait for the
in-band code to accept interrupts again. However, the interrupt
disable bit in the CPU would certainly be cleared in the meantime. For
this reason, depending on the interrupt type, the flow handlers as
modified by the pipeline code may have to mask the interrupt line
until the in-band handler has run from the root stage, lifting the
interrupt cause. This typically happens with level-triggered
interrupts, preventing the device from storming the CPU with a
continuous interrupt request.

Since all of the IRQ handlers sharing an interrupt line are either
in-band or out-of-band in a mutually exclusive way, such masking
cannot delay out-of-band events though.

> The pathological case of deferring level-triggered IRQs

```markdown
    /* root stage stalled on entry, no OOB handler */
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

The logic for adapting flow handlers dealing with any kind of
interrupt to pipelining can be decomposed in the following steps:

+ (optionally) enter the critical section protected by the IRQ
  descriptor lock, if the interrupt is shared among processors
  (e.g. device interrupts). If so, check if the interrupt handler may
  run on the current CPU (`irq_may_run()`). By definition, no locking
  would be required for per-CPU interrupts.

+ check whether we are entering the pipeline in order to deliver the
  interrupt to any out-of-band handler registered for
  it. `on_pipeline_entry()` returns a boolean value denoting this
  situation.

+ if on pipeline entry, we should pass the event on to the pipeline
  core by calling `handle_oob_irq()`. Upon return, this routine tells
  the caller whether any out-of-band handler was fired for the
  event.

  - if so, we may assume that the interrupt cause is now cleared in
    the device, and we may leave the flow handler, after having
    restored the interrupt line into a normal state. In case of a
    level-triggered interrupt which has been masked on entry to the
    flow handler, we need to unmask the line before leaving.

  - if no out-of-band handler was called, we should have performed any
    acknowledge and/or EOI to release the interrupt line, while
    leaving it masked if required before exiting the flow handler. In
    case of a level-triggered interrupt, we do want to leave it masked
    for solving the pathological case with interrupt deferral
    explained earlier.

+ if not on pipeline entry (i.e. second entry of the flow handler),
  then we must be running over the root stage, accepting interrupts,
  therefore we should fire the in-band handler(s) for the incoming
  event.

The original code for handling level-triggered interrupts is adapted
to interrupt pipelining according to the rules above, as follows:

> Example: adapting the handler dealing with level-triggered IRQs

```
--- a/kernel/irq/chip.c
+++ b/kernel/irq/chip.c
void handle_level_irq(struct irq_desc *desc)
{
	raw_spin_lock(&desc->lock);
	mask_ack_irq(desc);

 	if (!irq_may_run(desc))
 		goto out_unlock;
 
+	if (on_pipeline_entry()) {
+		if (handle_oob_irq(desc))
+			goto out_unmask;
+		goto out_unlock;
+	}
+
 	desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);
 
 	/*
@@ -642,7 +686,7 @@ void handle_level_irq(struct irq_desc *desc)
 
 	kstat_incr_irqs_this_cpu(desc);
 	handle_irq_event(desc);
-
+out_unmask:
 	cond_unmask_irq(desc);

out_unlock:
	raw_spin_unlock(&desc->lock);
}
```

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
spinlock]({{%relref "pipeline/usage/locking.md#new-spinlocks"
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

> Adapting the ARM GIC driver to interrupt pipelining

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

Due to the NMI-type nature of interrupts running out-of-band code,
such code might preempt in-band activities over the root stage in the
middle of a [critical section]({{%relref
"pipeline/optimistic.md#no-inband-reentry" %}}). For this reason, it
would be unsafe to call any in-band routine from an out-of-band
context.

However, we may schedule execution of in-band work handlers from
out-of-band code, using the regular `irq_work_queue()` service which
has been extended by the IRQ pipeline core. Such work request from the
head stage is scheduled for running over the root stage on the issuing
CPU as soon as the out-of-band activity quiesces on this processor. As
its name implies, the work handler runs in (in-band) interrupt
context.

{{% notice note %}}
The interrupt pipeline forces the use of a synthetic IRQ as a
notification signal for the IRQ work machinery, instead of a
hardware-specific interrupt vector. This special IRQ is labeled
_in-band work_ when reported by `/proc/interrupts`.
{{% /notice %}}
