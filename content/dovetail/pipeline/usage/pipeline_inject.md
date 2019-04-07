---
title: "IRQ injection"
date: 2018-07-01T15:38:34+02:00
---

## Sending out-of-band IPIs to remote CPUs {#oob-ipi}

Although the pipeline does not directly use IPIs internally, it
exposes two generic IPI vectors which autonomous cores may use in SMP
configuration for signaling the following events across CPUs:

- `RESCHEDULE_OOB_IPI`, the cross-CPU task reschedule request. This is
  available to the core's scheduler for kicking the task rescheduling
  procedure on remote CPUs, when the state of their respective
  runqueue has changed. For instance, a task sleeping on CPU #1 may be
  unblocked by a system call issued from CPU #0: in this case, the
  scheduler code running on CPU #0 is supposed to tell CPU #1 that it
  should reschedule. Typically, the EVL core does so from its
  `test_resched()` routine.

- `TIMER_OOB_IPI`, the cross-CPU timer reschedule request. Because
  software timers are in essence per-CPU beasts, this IPI is available
  to the core's timer management code for kicking the hardware timer
  programming procedure on remote CPUs, when the state of some
  software timer has changed. Typically, stopping a timer from a
  remote CPU, or migrating a timer from a CPU to another should
  trigger such signal. The EVL core does so from its
  `evl_program_remote_tick()` routine, which is called whenever the
  timer with the earliest timeout date enqueued on a remote CPU, may
  have changed.
 
As their respective name suggests, those two IPIs can be sent from
out-of-band context (as well as in-band), by calling the
`irq_pipeline_send_remote()` service.

---

{{< proto irq_pipeline_send_remote >}}
void irq_pipeline_send_remote(unsigned int ipi, const struct cpumask *cpumask)
{{< /proto >}}

{{% argument ipi %}}
The IPI number to send. There are only two legit values for this
argument: either RESCHEDULE_OOB_IPI, or TIMER_OOB_IPI. This is a
low-level service with not much parameter checking, so any other value
is likely to cause havoc.
{{% /argument %}}

{{% argument cpumask %}}
A CPU bitmask defining the target CPUs for the signal. The current CPU
is allowed to send a signal to itself, although this may not be the
faster path to running a local handler.
{{% /argument %}}

In order to receive these IPIs, an out-of-band handler must have been
set for them, mentioning the [IRQF_OOB flag]({{ < relref
"dovetail/pipeline/usage/irq_handling.md" >}}).

`irq_pipeline_send_remote()` serializes callers internally so that it
may be used from either stages: in-band or out-of-band.

---

## Injecting an IRQ event for the current CPU

In some very specific cases, we may need to inject an IRQ into the
pipeline by software as if such hardware event had happened on the
current CPU. `irq_inject_pipeline()` does exactly this.

---

{{< proto irq_inject_pipeline >}}
int irq_inject_pipeline(unsigned int irq)
{{< /proto >}}

{{% argument irq %}}
The IRQ number to inject. A valid interrupt descriptor must exist
for this interrupt.
{{% /argument %}}

`irq_inject_pipeline()` fully emulates the receipt of a hardware
event, which means that the common [interrupt pipelining logic]({{<
relref "dovetail/pipeline/porting/irqflow.md#genirq-flow" >}}) applies
to the new event:

- first, any [out-of-band handler]({{< relref
  "dovetail/pipeline/usage/irq_handling.md" >}}) is considered for
  delivery,

- then such event may be passed down the pipeline to the common
  in-band handler(s) in absence of out-of-band handler(s).

The pipeline priority rules apply accordingly:

- if the caller is in-band, _and_ an out-of-band handler is registered
  for the IRQ event, _and_ the out-of-band stage is [unstalled]({{<
  relref "dovetail/pipeline/optimistic.md" >}}), the execution stage
  is immediately switched to out-of-band for running the later, then
  restored to in-band before `irq_inject_pipeline()` returns.

- if the caller is out-of-band and there is no out-of-band handler,
  the IRQ event is deferred until the in-band stage resumes execution
  on the current CPU, at which point it is delivered to any in-band
  handler(s).

- in any case, should the current stage receive the IRQ event, the
  [virtual interrupt state]({{< relref
  "dovetail/pipeline/optimistic.md#virtual-i-flag" >}}) of that stage
  is always considered before deciding whether this event should be
  delivered immediately to its handler by `irq_inject_pipeline()`
  (_unstalled_ case), or deferred until the stage is unstalled
  (_stalled_ case).

This call returns zero on successful injection, or -EINVAL if the IRQ
has no valid descriptor.

---

## Direct logging of an IRQ event

Sometimes, running the full interrupt delivery logic
[`irq_inject_pipeline()`]({{< relref "#irq_inject_pipeline" >}})
implements for feeding an interrupt into the pipeline may be overkill,
because we can make assumptions about the current execution context,
and which stage should handle the event. The following fast helpers
can be used instead in this case:

---

{{< proto irq_post_inband >}}
void irq_post_inband(unsigned int irq)
{{< /proto >}}

{{% argument irq %}}
The IRQ number to inject into the in-band stage. A valid interrupt
descriptor must exist for this interrupt.
{{% /argument %}}

This routine may be used to mark an interrupt as pending directly into
the current CPU's log for the in-band stage. This is useful in either
of these cases:

- you know that the out-of-band stage is current, therefore this event
  has to be deferred until the in-band stage resumes on the current
  CPU later on. This means that you can simply post it to the in-band
  stage directly.

- you know that the in-band stage is current but [stalled]({{< relref
  "dovetail/pipeline/optimistic.md#virtual-i-flag" >}}), therefore
  this event can't be immediately delivered, so marking it as pending
  into the in-band stage is enough.

Interrupts must be [hard disabled]({{< relref
"dovetail/pipeline/usage/interrupt_protection##hard-irq-protection"
>}}) in the CPU before calling this routine.

---

{{< proto irq_post_oob >}}
void irq_post_oob(unsigned int irq)
{{< /proto >}}

{{% argument irq %}}
The IRQ number to inject into the out-of-band stage. A valid interrupt
descriptor must exist for this interrupt.
{{% /argument %}}

This routine may be used to mark an interrupt as pending directly into
the current CPU's log for the out-of-band stage. This is useful in
only one situation: you know that the out-of-band stage is current but
[stalled]({{< relref "dovetail/pipeline/optimistic.md#virtual-i-flag"
>}}), therefore this event can't be immediately delivered, so marking
it as pending into the out-of-band stage is enough.

Interrupts must be [hard disabled]({{< relref
"dovetail/pipeline/usage/interrupt_protection##hard-irq-protection"
>}}) in the CPU before calling this routine. If the out-of-band stage
is stalled as expected on entry to this helper, then interrupts must
be hard disabled in the CPU as well anyway.

---
