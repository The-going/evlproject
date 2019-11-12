---
title: "IRQ handling"
weight: 42
---

## Requesting an out-of-band IRQ {#request-oob-irq}

Dovetail introduces the new interrupt type flag `IRQF_OOB`, denoting
an out-of-band handler to the generic interrupt API routines:

- `setup_irq()` for early registration of special interrupts
- `request_irq()` for device interrupts
- `__request_percpu_irq()` for per-CPU interrupts

An IRQ action handler bearing this flag will run from out-of-band
context over the out-of-band stage, [regardless of the current
interrupt state]({{%relref
"dovetail/pipeline/_index.md#two-stage-pipeline" %}}) of the in-band
stage. If no out-of-band stage is present, the flag will be ignored,
with the interrupt handler running on the in-band stage as usual.

Conversely, out-of-band handlers can be dismissed using the usual
calls, such as:

- `free_irq()` for device interrupts
- `free_percpu_irq()` for per-CPU interrupts

Out-of-band IRQ handling has the following constraints:

- If the IRQ is shared, with multiple action handlers registered for
  the same event, all other handlers on the same interrupt channel
  must bear the `IRQF_OOB` flag too, or the request will fail.

{{% notice warning %}}
If meeting real-time requirements is your goal, sharing an IRQ line
among multiple devices operating from different execution stages
(in-band vs out-of-band) can only be a bad idea design-wise. You
should resort to this in desparate hardware situations **only**.
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

## Telling the companion kernel about entering, leaving the IRQ context {#irq-entry-exit}

Your companion core will most likely want to be notified each time a
new interrupt context is entered, typically in order to block any
further task rescheduling on its end. Conversely, this core will also
want to be notified when such context is exited, so that it can start
its rescheduling procedure, applying any change to the scheduler state
which occurred during the execution of the interrupt handler(s), such
as waking up a thread which was waiting for the incoming event.

To provide such support, Dovetail calls `irq_enter_pipeline()` on
entry to the pipeline when it receives an IRQ from the hardware, then
`irq_exit_pipeline()` right before it leaves the interrupt frame. It
defines empty placeholders for these hooks as follows, which are
picked in absence of a companion core in the kernel tree:

>  `linux/include/dovetail/irq.h` 
```
/* SPDX-License-Identifier: GPL-2.0 */
#ifndef _DOVETAIL_IRQ_H
#define _DOVETAIL_IRQ_H

/* Placeholders for pre- and post-IRQ handling. */

static inline void irq_enter_pipeline(void) { }

static inline void irq_exit_pipeline(void) { }

#endif /* !_DOVETAIL_IRQ_H */
```

As an illustration, the EVL core overrides the placeholders by
interposing the following file which comes earlier in the inclusion
order of C headers, providing its own set of hooks as follows:

> linux-evl/include/asm-generic/evl/irq.h
```
/* SPDX-License-Identifier: GPL-2.0 */
#ifndef _ASM_GENERIC_EVL_IRQ_H
#define _ASM_GENERIC_EVL_IRQ_H

#include <evl/irq.h>

static inline void irq_enter_pipeline(void)
{
#ifdef CONFIG_EVL
	evl_enter_irq();
#endif
}

static inline void irq_exit_pipeline(void)
{
#ifdef CONFIG_EVL
	evl_exit_irq();
#endif
}

#endif /* !_ASM_GENERIC_EVL_IRQ_H */
```

Eventually, the EVL core implements the `evl_enter_irq()` and
`evl_exit_irq()` routines in a final support header like this:

> linux-evl/include/evl/irq.h
```
/*
 * SPDX-License-Identifier: GPL-2.0
 *
 * Copyright (C) 2017 Philippe Gerum  <rpm@xenomai.org>
 */

#ifndef _EVL_IRQ_H
#define _EVL_IRQ_H

#include <evl/sched.h>

/* hard irqs off. */
static inline void evl_enter_irq(void)
{
	struct evl_rq *rq = this_evl_rq();

	rq->local_flags |= RQ_IRQ;
}

/* hard irqs off. */
static inline void evl_exit_irq(void)
{
	struct evl_rq *this_rq = this_evl_rq();

	this_rq->local_flags &= ~RQ_IRQ;

	/*
	 * CAUTION: Switching stages as a result of rescheduling may
	 * re-enable irqs, shut them off before returning if so.
	 */
	if ((this_rq->flags|this_rq->local_flags) & RQ_SCHED) {
		evl_schedule();
		if (!hard_irqs_disabled())
			hard_local_irq_disable();
	}
}

#endif /* !_EVL_IRQ_H */
```
