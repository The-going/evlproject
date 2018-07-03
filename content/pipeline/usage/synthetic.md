---
title: "Synthetic IRQs"
date: 2018-06-27T09:55:57+02:00
weight: 3
draft: false
---

The pipeline introduces an additional type of interrupts, which are
purely software-originated, with no hardware involvement. These IRQs
can be triggered by any kernel code. Synthetic IRQs are inherently
per-CPU events. Because the common pipeline flow applies to synthetic
interrupts, it is possible to attach them to out-of-band and/or
in-band handlers, just like device interrupts.

{{% notice note %}}
Synthetic interrupts and regular softirqs differ in essence: the
latter only exist in the in-band context, and therefore cannot trigger
out-of-band activities.
{{% /notice %}}

Synthetic interrupt vectors are allocated from the
*synthetic_irq_domain*, using the `irq_create_direct_mapping()`
routine.

For instance, a synthetic interrupt can be used for triggering an
in-band activity on the root stage from the head stage as follows:

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

Code can schedule the execution of `sirq_handler()` from the
out-of-band context in two different ways:

> Using the common injection service

```markdown
	irq_pipeline_inject(sirq);
```

> Using the lightweight injection method (requires interrupts to be
  disabled in the CPU)

```markdown
	unsigned long flags = hard_local_irqsave();
	irq_stage_post_root(sirq);
	hard_local_irqrestore(flags);
```

{{% notice tip %}}
Assuming that no interrupt may be pending in the event log for the
head stage at the time this code runs, the method above relies on the
invariant that in a pipeline interrupt model, IRQs pending for the
root stage will have to wait for the head stage to quiesce before they
can be handled. Therefore, it is pointless to try synchronizing the
interrupts pending for the root stage from the head stage, which the
`irq_pipeline_inject()` service would do systematically.
`irq_stage_post_root()` simply marks the event as pending in the event
log of the root stage for the current CPU, then returns. This event
would be played as a result of synchronizing the log automatically when
the current CPU switches back to the root stage.
{{% /notice %}}

Conversely, a synthetic interrupt can be handled from the out-of-band
context:

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
  
Code can trigger the immediate execution of `sirq_oob_handler()` on
the head stage as follows:

```markdown
	irq_pipeline_inject(sirq);
```

{{% notice note %}}
Calling `irq_stage_post_head(sirq)` from the root stage to trigger an
out-of-band event is most often not the right way to do this, because
this service would not synchronize the interrupt log before
returning. In other words, the `sirq` event would still be pending for
the head stage despite the fact that it should have preempted the root
stage before returning to the caller.
{{% /notice %}}
