---
title: "Synthetic IRQs"
weight: 45
---

The pipeline introduces an additional type of interrupts, which are
purely software-originated, with no hardware involvement. These IRQs
can be triggered by any kernel code. Synthetic IRQs are inherently
per-CPU events. Because the common pipeline flow applies to synthetic
interrupts, it is possible to attach them to out-of-band and/or
in-band handlers, just like device interrupts.

Synthetic interrupts abide by the normal rules with respect to
interrupt masking: such IRQs may be deferred until the stage they
should be handled from is unstalled.

{{% notice note %}}
Synthetic interrupts and softirqs differ in essence: the
latter only exist in the in-band context, and therefore cannot trigger
out-of-band activities. Synthetic interrupts used to be called
_virtual_ IRQs (or _virq_ for short) by the legacy I-pipe
implementation, Dovetail's ancestor; such rename clears the confusion
with the way abstract interrupt numbers defined within interrupt
domains may be called elsewhere in the kernel code base (i.e.
_virtual interrupts_ too).
{{% /notice %}}

## Allocating synthetic interrupts

Synthetic interrupt vectors are allocated from the
*synthetic_irq_domain*, using the `irq_create_direct_mapping()`
routine.

A synthetic interrupt handler can be installed for running on the in-band
stage upon a scheduling request (i.e. being posted) from an
out-of-band context as follows:

```markdown
#include <linux/irq_pipeline.h>

static irqreturn_t sirq_handler(int sirq, void *dev_id)
{
	do_in_band_work();

	return IRQ_HANDLED;
}

static struct irqaction sirq_action = {
        .handler = sirq_handler,
        .name = "In-band synthetic interrupt",
        .flags = IRQF_NO_THREAD,
};

unsigned int alloc_sirq(void)
{
	unsigned int sirq;

	sirq = irq_create_direct_mapping(synthetic_irq_domain);
	if (!sirq)
		return 0;
	
	setup_percpu_irq(sirq, &sirq_action);

	return sirq;
}
```

A synthetic interrupt handler can be installed for running from the
oob stage upon a trigger from an in-band context as follows:

```markdown
static irqreturn_t sirq_oob_handler(int sirq, void *dev_id)
{
	do_out_of_band_work();

	return IRQ_HANDLED;
}

unsigned int alloc_sirq(void)
{
	unsigned int sirq;

	sirq  = irq_create_direct_mapping(synthetic_irq_domain);
	if (!sirq)
		return 0;
     
	ret = __request_percpu_irq(sirq, sirq_oob_handler,
                                   IRQF_OOB,
                                   "Out-of-band synthetic interrupt",
                                   dev_id);
	if (ret) {
        	irq_dispose_mapping(sirq);
		return 0;
	}

	return sirq;
}
```
  
## Scheduling in-band execution of a synthetic interrupt handler

The execution of `sirq_handler()` in the in-band context can be
scheduled (or posted) from the out-of-band context in two different
ways:

### Using the common injection service

```markdown
	irq_pipeline_inject(sirq);
```

### Using the lightweight injection method (requires interrupts to be disabled in the CPU)

```markdown
	unsigned long flags = hard_local_irqsave();
	irq_post_inband(sirq);
	hard_local_irqrestore(flags);
```

{{% notice tip %}}
Assuming that no interrupt may be pending in the event log for the
oob stage at the time this code runs, the second method relies on the
invariant that in a pipeline interrupt model, IRQs pending for the
in-band stage will have to wait for the oob stage to quiesce before they
can be handled. Therefore, it is pointless to check for synchronizing the
interrupts pending for the in-band stage from the oob stage, which the
`irq_pipeline_inject()` service would do systematically.
`irq_post_inband()` simply marks the event as pending in the event
log of the in-band stage for the current CPU, then returns. This event
would be played as a result of synchronizing the log automatically when
the current CPU switches back to the in-band stage.
{{% /notice %}}

It is also valid to post a synthetic interrupt to be handled on the
in-band stage from an in-band context, using
`irq_pipeline_inject()`. In such a case, the normal rules of interrupt
delivery apply, depending on the state of the [virtual interrupt
disable flag]({{%relref
"dovetail/pipeline/_index.md#virtual-i-flag" %}}) for the in-band
stage: the IRQ is immediately delivered, with the call to
`irq_pipeline_inject()` returning only after the handler has run.

## Triggering out-of-band execution of a synthetic interrupt handler

Conversely, the execution of `sirq_handler()` on the oob stage can be
triggered from the in-band context as follows:

```markdown
	irq_pipeline_inject(sirq);
```

Since the oob stage has precedence over the in-band stage for execution
of any pending event, this IRQ is immediately delivered, with the call
to `irq_pipeline_inject()` returning only after the handler has run.

It is also valid to post a synthetic interrupt to be handled on the
oob stage from an out-of-band context, using
`irq_pipeline_inject()`. In such a case, the normal rules of interrupt
delivery apply, depending on the state of the virtual interrupt
disable flag [for the oob stage]({{%relref
"dovetail/pipeline/interrupt_protection.md#oob-stall-flag" %}}).

{{% notice note %}}
Calling `irq_post_oob(sirq)` from the in-band stage to trigger an
out-of-band event is most often not the right way to do this, because
this service would not synchronize the interrupt log before
returning. In other words, the `sirq` event would still be pending for
the oob stage despite the fact that it should have preempted the in-band
stage before returning to the caller.
{{% /notice %}}

---

{{<lastmodified>}}
