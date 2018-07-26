---
title: "Developer's Notes"
date: 2018-07-03T19:32:57+02:00
weight: 8
draft: false
---

## Generic

### Fundamentally preemption-safe contexts

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

### Checking for out-of-band interrupt property

The `IRQF_OOB` action flag should **NOT** be used for testing whether
an interrupt is out-of-band, because out-of-band handling may be
turned on/off dynamically on an IRQ descriptor using
`irq_switch_oob()`, which would not translate to `IRQF_OOB` being
set/cleared for the attached action handlers.

`irq_is_oob()` is the right way to check for out-of-band handling.

### stop_machine() hard disables interrupts

The regular stop_machine() services guarantees that all online CPUs
are spinning non-preemptible in a known code location before a subset
of them may safely run a stop-context function. This service is
typically useful for live patching the kernel code, or changing global
memory mappings, so that no activity could run in parallel until the
system has returned to a stable state after all stop-context
operations have completed.
    
When interrupt pipelining is enabled, Dovetail provides the same
guarantee by restoring hard interrupt disabling where virtualizing the
interrupt disable flag would defeat it.

As those lines are written, all stop_machine() use cases must also
exclude any head stage activity (e.g. ftrace live patching the kernel
code for installing tracepoints), or happen before any such activity
can ever take place (e.g. KPTI boot mappings). This is a basic
assumption: stop_machine() could not get in the way of
latency-sensitive processes, simply because the latter could not keep
running safely until a call to the former has completed anyway.

However, one should keep an eye on stop_machine() upstream,
identifying new callers which might cause unwanted latency spots under
specific circumstances (maybe even abusing the interface).

## ARM

### Context assumption with outer L2 cache

There is no reason for the outer cache to be
invalidated/flushed/cleaned from an out-of-band context, all cache
maintenance operations must happen from in-band code. Therefore, we
neither need nor want to convert the spinlock serializing access to
the cache maintenance operations for L2 to a hard lock.

{{% notice tip %}}
Conversion to hard lock may have cause latency to skyrocket on some
i.MX6 hardware, equipped with PL22x cache units, or PL31x with errata
588369 or 727915 for particular hardware revisions, as each background
operation was awaited for completion for working around some silicon
bug, with hard irqs disabled.
{{% /notice %}}
