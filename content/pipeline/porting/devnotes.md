---
title: "Developer's Notes"
date: 2018-07-03T19:32:57+02:00
weight: 10
draft: false
---

## Intrinsically preemption-safe contexts

Over a few contexts, we may traverse code using unprotected,
preemption-sensitive accessors such as `percpu()` without disabling
preemption specifically, because either one condition is true;

- if `preempt_count()` bears either of the `PIPELINE_MASK` or
  `STAGE_MASK` bits, which turns preemption off, therefore CPU
  migration cannot happen (`debug_smp_processor_id()` and preempt
  checks in `percpu` accessors would detect such context properly
  too).

- if we are running over the context of the root stage's event log
  syncer (`irq_stage_sync_current()`) playing a deferred interrupt, in
  which case the virtual interrupt disable bit is set, so no CPU
  migration may occur either.

For instance, the following contexts qualify:

- `clockevents_handle_event()`, which should either be called from the
  head stage - therefore `STAGE_MASK` is set - when the [proxy tick
  device is active] ({{% relref
  "/pipeline/porting/timer.md#proxy-tick-logic" %}}) on the CPU,
  and/or from the root stage playing a timer interrupt event from the
  corresponding device.

- any IRQ flow handler _from kernel/irq/chip.c_. When called from
  `generic_pipeline_irq()` for pushing an external event to the
  pipeline, `on_pipeline_entry()` is true, which indicates that
  PIPELINE_MASK is set. When called for playing a deferred interrupt
  on the root stage, the virtual interrupt disable bit is set.

## Checking for out-of-band interrupt property

The `IRQF_OOB` action flag should **NOT** be used for testing whether
an interrupt is out-of-band, because out-of-band handling may be
turned on/off dynamically on an IRQ descriptor using
`irq_switch_oob()`, which would not translate to `IRQF_OOB` being
set/cleared for the attached action handlers.

`irq_is_oob()` is the right way to check for out-of-band handling.
