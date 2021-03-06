---
title: "Developer's Notes"
weight: 101
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

- if we are running over the context of the in-band stage's event log
  syncer (`sync_current_stage()`) playing a deferred interrupt, in
  which case the virtual interrupt disable bit is set, so no CPU
  migration may occur either.

For instance, the following contexts qualify:

- `clockevents_handle_event()`, which should either be called from the
  oob stage - therefore `STAGE_MASK` is set - when the [proxy tick
  device is active] ({{% relref
  "dovetail/porting/timer#proxy-tick-logic" %}}) on the CPU,
  and/or from the in-band stage playing a timer interrupt event from the
  corresponding device.

- any IRQ flow handler _from kernel/irq/chip.c_. When called from
  `generic_pipeline_irq()` for pushing an external event to the
  pipeline, `on_pipeline_entry()` is true, which indicates that
  PIPELINE_MASK is set. When called for playing a deferred interrupt
  on the in-band stage, the virtual interrupt disable bit is set.

### Checking for out-of-band interrupt property

The `IRQF_OOB` action flag should _not_ be used for testing whether
an interrupt is out-of-band, because out-of-band handling may be
turned on/off dynamically on an IRQ descriptor using
`irq_switch_oob()`, which would not translate to `IRQF_OOB` being
set/cleared for the attached action handlers.

`irq_is_oob()` is the right way to check for out-of-band handling.

### `stop_machine()` hard disables interrupts

The `stop_machine()` service guarantees that all online CPUs are
spinning non-preemptible in a known code location before a subset of
them may safely run a stop-context function. This service is typically
useful for live patching the kernel code, or changing global memory
mappings, so that no activity could run in parallel until the system
has returned to a stable state after all stop-context operations have
completed.
    
When interrupt pipelining is enabled, Dovetail provides the same
guarantee by restoring hard interrupt disabling where virtualizing the
interrupt disable flag would defeat it.

As those lines are written, all `stop_machine()` use cases must also
exclude any oob stage activity (e.g. ftrace live patching the kernel
code for installing tracepoints), or happen before any such activity
can ever take place (e.g. KPTI boot mappings). Dovetail makes a basic
assumption that `stop_machine()` could not get in the way of
latency-sensitive processes, simply because the latter could not keep
running safely until a call to the former has completed anyway.

However, one should keep an eye on `stop_machine()` usage upstream,
identifying new callers which might cause unwanted latency spots under
specific circumstances (maybe even abusing the interface).

### Virtual interrupt disable state breakage

When some WARN_ON() triggers due to a wrong interrupt disable state
(e.g. entering the softirqs/bh code with IRQs unexpectedly [virtually]
disabled), this may be due to the CPU and virtual interrupt states
being out-of-sync when traversing the epilogue code after a syscall,
IRQ or trap has been handled during the latest kernel entry.

Typically, do_work_pending() or do_notify_resume() should make sure to
reconcile both states in the work loop, and also to restore the
virtual state they received on entry before returning to their caller.

{{% notice tip %}}
The routines just mentioned always enter from their assembly call site
with interrupts hard disabled in the CPU. However, they may be entered
with the virtual interrupt state enabled or disabled, depending on the
kind of event which led to them eventually. Typically, a system call
epilogue would always enter with the virtual state enabled, but a
fault might also occur when the virtual state is disabled though. The
epilogue routine called for finalizing some IRQ handling must enter with
the virtual state enabled, since the latter is a pre-requisite for
running such code.
{{% /notice %}}

### Losing the timer tick

The symptom of a common issue in a Dovetail port is losing the timer
interrupt when the autonomous core takes control over the [tick
device]({{% relref "dovetail/porting/timer.md" %}}), causing
the in-band kernel to stall. After some time spent hanging, the
in-band kernel may eventually complain about a RCU stall situation
with a message like `INFO: rcu_preempt detected stalls on CPUs/tasks`
followed by stack dump(s). In other cases, the machine may simply lock
up due to an interrupt storm.

This is typical of timer interrupt events not flowing down normally to
the in-band kernel anymore because something went wrong as soon as the
proxy tick device replaced the regular device for serving in-band
timing requests. When this happens, you should check the following code
spots for bugs:

- the timer acknowledge code is wrong once called from the [oob
  stage]({{%relref "dovetail/pipeline/_index.md#two-stage-pipeline"
  %}}), which is going to be the case as soon as an autonomous core
  installs the [proxy tick device]({{% relref
  "dovetail/porting/timer.md" %}}) for interposing on the
  timer. Being wrong here means performing actions which are not legit
  from [such a context]({{% relref
  "dovetail/rulesofthumb#safe-inband-code" %}}).

- the _irqchip_ driver managing the interrupt event for the timer tick
  is wrong somehow, causing such interrupt to stay masked or stuck for
  some reason whenever it is switched to [out-of-band mode]({{% relref
  "dovetail/pipeline/irq_handling.md" %}}). You need to
  double-check the implementation of the [chip handlers]({{% relref
  "dovetail/porting/irqflow#irqchip-fixups" %}}),
  considering the effects and requirements of interrupt pipelining.

- power management (CONFIG_CPUIDLE) gets in the way, often due to the
  infamous C3STOP misfeature turning off the original timer hardware
  controlled by the proxy device. A detailed explanation is given in
  Documentation/irq_pipeline.rst when discussing the few changes to
  the scheduler core for supporting the Dovetail interface. If this is
  acceptable from a power saving perspective, having the autonomous
  core prevent the in-band kernel from entering a deeper C-state is
  enough to fix the issue, by overriding the `irq_cpuidle_control()`
  routine as follows:

```
bool irq_cpuidle_control(struct cpuidle_device *dev,
			 struct cpuidle_state *state)
{
	/*
	 * Deny entering sleep state if this entails stopping the
	 * timer (i.e. C3STOP misfeature).
	 */
	if (state && (state->flags & CPUIDLE_FLAG_TIMER_STOP))
		return false;

	return true;
}
```

{{% notice tip %}}
Printk-debugging such timer issue *requires* enabling [raw
printk()]({{% relref "dovetail/porting/rawprintk.md" %}}) support,
you won't get away with tracing the kernel behavior using the plain
`printk()` routine for this, because most of the output would remain
stuck into a buffer, never reaching the console driver before the
board hangs eventually.
{{% /notice %}}

### Hard interrupt masking in clock chip handlers

The only valid way of sharing a clock tick device between the in-band
and out-of-band stages is to access it through [the tick
proxy]({{%relref "dovetail/porting/timer.md" %}}). For this reason, we
don't need to enforce [hard interrupt masking]({{%relref
"dovetail/pipeline/interrupt_protection.md" %}}) in clock chip
handlers to make them pipeline-safe, because once proxying is active
for a tick device, hardware interrupts are off across calls to its
handlers when applicable. As a result, either all accesses to the
clock chip handlers are proxied and proper masking is already in
place, or there is no proxy, which means the in-band kernel is still
controlling the device, in which case there is no way we might
conflict with out-of-band accesses.

Obviously, this fact does not preclude why we would still want to
serialize CPUs when accessing shared data there, in which case [hard
locking]({{%relref "dovetail/pipeline/locking.md" %}}) should be in
place to ensure this.

### Make no assumption in virtualizing arch_local_irq_restore()

Do not make any assumption with respect to the current interrupt state
when `arch_local_irq_restore()` is called, specifically don't expect
the inband stage to be stalled on entry. Some archs use constructs
like follows, which breaks such assumption:

```
	local_save_flags(flags);
	local_irq_enable();
	...
	local_irq_restore(flags);
```

In that case, we do want the stall bit to be restored unconditionally
from _flags_. A correct implementation would be:

```
static inline notrace void arch_local_irq_restore(unsigned long flags)
{
	inband_irq_restore(arch_irqs_disabled_flags(flags));
	barrier();
}

```

### RCU and out-of-band context

The out-of-band context is semantically equivalent to the NMI context,
therefore the current CPU cannot be in an extended quiescent state
RCU-wise if running oob. CAUTION: the converse assertion is **NOT**
true (i.e. a CPU running code on the in-band stage may be idle
RCU-wise).

### Common services which are safe in out-of-band context

Dovetail guarantees that the following services are safe to call from
the out-of-band stage:

- irq_work() may be called from the out-of-band stage to schedule a
  (synthetic) interrupt in the in-band stage.

- __raise_softirq_irqoff() may be called from the out-of-band stage to
  schedule a softirq. For instance, the EVL network stack uses this to
  kick the NET_TX_SOFTIRQ event in the in-band stage.

- printk() may be called from any context, including out-of-band
  interrupt handlers. Messages are queued, then passed to the output
  device(s) only when the in-band stage resumes though.

## ARM

### Context assumption with outer L2 cache

There is no reason for the outer cache to be
invalidated/flushed/cleaned from an out-of-band context, all cache
maintenance operations must happen from in-band code. Therefore, we
neither need nor want to convert the spinlock serializing access to
the cache maintenance operations for L2 to a hard lock.

{{% notice note %}}
This above assumption is unfortunately only partially right, because
at some point in the future we may want to run DMA transfers from the
out-of-band context, which could entail cache maintenance operations.
{{% /notice %}}

Conversion to hard lock may cause latency to skyrocket on some i.MX6
hardware, equipped with PL22x cache units, or PL31x with errata 588369
or 727915 for particular hardware revisions, as each background
operation would be awaited for completion with hard irqs disabled, in
order to work around some silicon bug.

---

{{<lastmodified>}}
