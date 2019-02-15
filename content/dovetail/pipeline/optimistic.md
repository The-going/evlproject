---
title: "Optimistic Interrupt Protection"
menuTitle: "Optimistic protection"
date: 2018-06-27T10:08:28+02:00
weight: 1
draft: false
---

Predictable response time of out-of-band handlers to IRQ receipts
requires the in-band kernel work not to be allowed to delay them by
masking interrupts in the CPU.

However, critical sections delimited this way by the in-band code must
still be enforced for the *in-band stage*, so that system integrity is
not at risk. This means that although out-of-band IRQ handlers may run
at any time while the *oob stage* is accepting interrupts, in-band IRQ
handlers should be allowed to run only when the in-band stage is
accepting interrupts too. So we need to decouple the interrupt masking
and delivery logic which applies to the oob stage from the one in
effect on the in-band stage, by implementing a dual interrupt control.

## Virtual interrupt disabling {#virtual-i-flag}

To this end, a software logic managing a virtual interrupt disable
flag is introduced by the interrupt pipeline between the hardware and
the generic IRQ management layer. This logic can mask IRQs from the
perspective of the in-band kernel work when `local_irq_save()`,
`local_irq_disable()` or any lock-controlled masking operations like
`spin_lock_irqsave()` is called, while still accepting IRQs from the
CPU for immediate delivery to out-of-band handlers.

When a real IRQ arrives while interrupts are virtually masked, the
event is logged for the receiving CPU, kept there until the virtual
interrupt disable flag is cleared at which point it is dispatched as
if it just happened. The principle of deferring interrupt delivery
based on a software flag coupled to an event log has been originally
described as *Optimistic interrupt protection* in [this
paper](https://www.usenix.org/legacy/publications/library/proceedings/micro93/full_papers/stodolsky.txt).
It was originally intended as a low-overhead technique for
manipulating the processor interrupt state, reducing the cost of
interrupt masking for the common case of absence of interrupts.

In Dovetail's two-stage pipeline, the oob stage protects from
interrupts by disabling them in the CPU's status register as usual,
while the in-band stage disables interrupts only virtually. A stage for
which interrupts are disabled is said to be *stalled*. Conversely,
*unstalling* a stage means re-enabling interrupts for it.

{{% notice warning %}}
Obviously, stalling the oob stage implicitly means disabling
further IRQ receipts for the in-band stage down the pipeline too.
{{% /notice %}}

## Interrupt deferral for the *in-band stage*

When the in-band stage is stalled because the virtual interrupt disable
flag is set, any IRQ event which was not immediately delivered to the
*oob stage* is recorded into a per-CPU log, postponing delivery to the
in-band kernel handler.

Such delivery is deferred until the in-band kernel code clears the
virtual interrupt disable flag by calling `local_irq_enable()` or any
of its variants, which unstalls the in-band stage. When this happens, the
interrupt state is resynchronized by playing the log, firing the
in-band handlers for which an IRQ event is pending.

```markdown
   /* Both stages unstalled on entry */
   local_irq_save(flags);
   <IRQx received: no out-of-band handler>
       (pipeline logs IRQx event)
   ...
   local_irq_restore(flags);
       (pipeline plays IRQx event)
            handle_IRQx_interrupt();
```
        
If the in-band stage is unstalled at the time of the IRQ receipt, the
in-band handler is immediately invoked, just like with the
non-pipelined IRQ model.

## All interrupts are (seemingly) NMIs {#no-inband-reentry}

From the standpoint of the in-band kernel code (i.e. the one running
over the *in-band* interrupt stage) , the interrupt pipelining logic
virtually turns all device IRQs into NMIs, for running out-of-band
handlers.

For this reason, out-of-band code may generally **NOT** re-enter
in-band code, for preventing creepy situations like this one:

```markdown
   /* in-band context */
   spin_lock_irqsave(&lock, flags);
      <IRQx received: out-of-band handler installed>
         handle_oob_event();
            /* attempted re-entry to in-band from out-of-band. */
            in_band_routine();
               spin_lock_irqsave(&lock, flags);
               <DEADLOCK>
               ...
            ...
         ...
   ...
   spin_unlock irqrestore(&lock, flags);
```

Even in absence of an attempt to get a spinlock recursively, the outer
in-band code in the example above is entitled to assume that no access
race can occur on the current CPU while interrupts are
masked. Re-entering in-band code from an out-of-band handler would
invalidate this assumption.

In rare cases, we may need to fix up the in-band kernel routines in
order to allow out-of-band handlers to call them. Typically, atomic
helpers are such routines, which serialize in-band and out-of-band
callers.

For all other cases, the [IRQ work API]({{%relref
"dovetail/pipeline/porting/irqflow.md#irq-work" %}}) is available for
scheduling the execution of a routine from the oob stage, which will
be invoked later from the in-band stage as soon as it gets back in
control on the current CPU.
