---
title: "IRQ handling"
date: 2018-07-01T14:05:58+02:00
weight: 2
draft: false
---

The driver API to the IRQ subsystem exposes the new interrupt type
flag `IRQF_OOB`, denoting an out-of-band handler with the
following routines:

- `setup_irq()` for early registration of special interrupts
- `request_irq()` for device interrupts
- `__request_percpu_irq()` for per-CPU interrupts

An IRQ action handler bearing this flag will run from out-of-band
context over the oob stage, [regardless of the current interrupt
state]({{%relref "dovetail/pipeline/_index.md#two-stage-pipeline" %}})
of the in-band stage. If no oob stage is present, the flag will be
ignored, with the interrupt handler running in-band over the in-band
stage as usual.

Conversely, out-of-band handlers can be dismissed using the regular
API, such as:

- `free_irq()` for device interrupts
- `free_percpu_irq()` for per-CPU interrupts

Out-of-band IRQ handling has the following constraints:

- If the IRQ is shared, with multiple action handlers registered for
  the same event, all other handlers on the same interrupt channel
  must bear the `IRQF_OOB` flag too, or the request will fail.

{{% notice warning %}}
If meeting real-time requirements is your goal, sharing an IRQ line
among multiple devices can only be a bad idea. You may want to do
that in desparate hardware situations **only**.
{{% /notice %}}

- Obviously, out-of-band handlers cannot be threaded (`IRQF_NO_THREAD`
  is implicit, `IRQF_ONESHOT` is ignored).

> Installing an out-of-band handler for a device interrupt

```
#include <linux/interrupt.h>

static irqreturn_t oob_interrupt_handler(int irq, void *dev_id)
{
	...
	return IRQ_HANDLED;
}

init __init driver_init_routine(void)
{
	int ret;

	...
	ret = request_irq(DEVICE_IRQ, oob_interrupt_handler,
			  IRQF_OOB, "Out-of-band device IRQ",
			  device_data);
	if (ret)
		goto fail;

	return 0;
fail:
	/* Unwind upon error. */
	...
}
```
