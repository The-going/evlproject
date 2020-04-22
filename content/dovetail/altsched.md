---
title: "Alternate scheduling"
weight: 119
---

For [specific use cases requiring reliable, ultra-low response
times]({{< relref "dovetail/_index.md#dual-kernel-upsides" >}}), we
want to enable hosted autonomous software cores to control common
Linux tasks based on their own scheduler infrastructure, fully
decoupled from the host's scheduler, with absolute priority over all
other kernel activities.

This being said, Dovetail also promotes the idea that a *dual kernel*
system should keep the functional overlap between the main kernel and
the autonomous core minimal. To this end, a task from such core should
be merely seen as a regular Linux task with additional scheduling
capabilities guaranteeing very low and bounded response times. To
support such idea, Dovetail enables kthreads and regular user tasks to
run alternatively in the out-of-band execution context introduced by
the interrupt pipeline (aka *out-of-band* stage), or the common
in-band kernel context for GPOS operations (aka *in-band*
stage). These new capabilities are built on the [interrupt
pipeline]({{< relref "dovetail/pipeline/_index.md" >}}) machinery.

As a result, autonomous core applications in user-space benefit from
the common Linux programming model - including virtual memory
protection -, and still have access to the main kernel services when
carrying out non time-critical work.

Dovetail provides mechanisms to autonomous cores for supporting this
as follows:

- services for moving Linux tasks back and forth between the in-band
  and out-of-band stages in an orderly and safe way, properly
  synchronizing the operation between the two schedulers involved.

- notifications about in-band events the autonomous core may want to
  know about for maintaining its own version of the current task's
  state.

- notifications about events such as faults, syscalls and other
  exceptions issued from the out-of-band execution stage.

- integrated support for performing context switches between
  out-of-band tasks, including memory context and FPU management.

## Theory of operations {#altsched-theory}

- a Linux task running in user-space, or a kernel thread, need to
  initialize the alternate scheduling feature for themselves with a
  call to [dovetail_init_altsched()]({{< relref
  "#dovetail_init_altsched" >}}). For instance, the EVL core does so
  as part of the attachment process of a thread to the autonomous core
  when [evl_attach_self()]({{% relref
  "core/user-api/thread/_index.md#evl_attach_self" %}}) is called by
  the application.

- once the alternate scheduling feature is initialized, the task
  should enable it by a call to [dovetail_start_altsched()]({{< relref
  "#dovetail_start_altsched" >}}). From this point, that task:

  * can switch between the in-band and out-of-band execution stages
    freely.

  * can emit out-of-band system calls the autonomous core can handle.

  * is tracked by Dovetail's notification system, so that in-band and
    out-of-band events which may involve such task are dispatched to
    the core.

  * can be involved in Dovetail-based context switches triggered by
    the autonomous core, independently from the scheduling operations
    carried out by the main kernel.

    Conversely, [dovetail_stop_altsched()]({{< relref
    "#dovetail_stop_altsched" >}}) disables the notifications for a
    task, which is likely to be detached from the autonomous core
    later on.

- at any point in time, any Linux task is either controlled by the
  main kernel or by the autonomous core, scheduling-wise. There cannot
  be any overlap (for obvious reasons):

  * a task which is currently scheduled by the autonomous core runs or
    sleeps on the out-of-band stage. At the same time, such task is
    deemed to be sleeping in `TASK_INTERRUPTIBLE` state for the main
    kernel.

  * conversely, a task which is controlled by the main kernel runs or
    sleeps on the in-band stage. It must be considered as blocked on
    the other side. For instance, the EVL core defines the `T_INBAND`
    blocking condition for representing such state.

- since the out-of-band stage receives interrupts first, regardless of
  any interrupt masking which may be in effect for the ongoing in-band
  work, the autonomous core can run [high-priority interrupt
  handlers]({{% relref
  "dovetail/pipeline/_index.md#two-stage-pipeline" %}}) then
  reschedule with no delay.

- tasks may switch back and forth between the in-band and out-of-band
  stages at will. However, this comes at a cost:

  * the process of switching stages is heavyweight, it includes a
    double scheduling operation at least (i.e. one to suspend on the
    exited stage, one to resume from the opposite stage).

  * **only** the out-of-band stage guarantees bounded response times
    to external and internal events. Therefore, a task which leaves
    the out-of-band stage for resuming in-band looses such guarantee,
    until it has fully switched back to out-of-band context at some
    point later.

- at any point in time, the autonomous core keeps the CPU busy until
  no more task it knows about is runnable on that CPU, on the
  out-of-band stage. When the out-of-band activity quiesces, the core
  is expected to relinquish the CPU to the in-band stage by scheduling
  in the context which was originally preempted. This is commonly done
  by having a task placeholder with the lowest possible priority
  represent the main kernel and its in-band context linked to each
  per-CPU run queue maintained by the autonomous core. For instance,
  the EVL core assigns such a placeholder task to its [SCHED_IDLE
  policy]({{< relref "core/user-api/scheduling/_index.md#SCHED_IDLE"
  >}}), which get picked when other policies have no runnable task
  on the CPU.

- once the in-band context resumes, interrupt events which have no
  out-of-band handlers are delivered to the regular in-band IRQ
  handlers installed by the main kernel, if the [virtual masking
  state]({{% relref "dovetail/pipeline/_index.md#virtual-i-flag"
  %}}) allows it.

![Alt text](/images/altsched.png "Alternate scheduling")

---

{{< proto dovetail_init_altsched >}}
void dovetail_init_altsched(struct dovetail_altsched_context *p)
{{< /proto >}}

This call initializes the alternate scheduling context for the current
task; this should be done once, before the task calls
[dovetail_start_altsched()]({{< relref "#dovetail_start_altsched" >}}).

{{% argument p %}}
The alternate scheduling context is kept in a per-task structure of
type _dovetail_altsched_context_ which should be maintained by the
autonomous core. This can be done as part of the [per-task context
management]({{< relref "#dovetail-task-context" >}}) feature
Dovetail introduces.
{{% /argument %}}

---

{{< proto dovetail_start_altsched >}}
void dovetail_start_altsched(void)
{{< /proto >}}

This call tells the kernel that the current task may request alternate
scheduling operations any time from now on, such as [switching
out-of-band]({{< relref "#oob-switch" >}}) or back [in-band]({{<
relref "#inband-switch" >}}). It also activates the [event
notifier]({{< relref "#event-notifier" >}}) for the task, which allows
it to emit out-of-band system calls to the core.

---

{{< proto dovetail_stop_altsched >}}
void dovetail_stop_altsched(void)
{{< /proto >}}

This call disables the [event notifier]({{< relref "#event-notifier"
>}}) for the current task, which must be done before dismantling the
alternate scheduling support for that task in the autonomous core.

---

## What you really need to know at this point

There is a not-so-subtle but somewhat confusing distinction between
_running a Linux task out-of-band_, and running whatever code from the
_out-of-band execution stage_.

- In the first case, the task is not controlled by the main kernel
  scheduler anymore, but runs under the supervision of the autonomous
  core. This is obtained by performing an [out-of-band switch]({{<
  relref "#oob-switch" >}}) for a task.

- In the other case, the current underlying context (task or whatever
  else) may or may not be controlled by the main kernel, but the
  interrupt pipeline machinery has switched the CPU to the out-of-band
  execution mode, which means that only interrupts bearing the
  IRQF_OOB flag are delivered. Typically, [run_oob_call()]({{% relref
  "dovetail/pipeline/stage_escalation.md" %}}) is a service provided
  by the interrupt pipeline which executes a function call over this
  context, without requiring the calling task to be scheduled by the
  autonomous core.

You will also find references to pseudo-routines called
`core_schedule()`, `core_suspend_task()` or `core_resume_task()` in
various places. Don't look for them into the code, they don't actually
exist: those routines are mere placeholders for the corresponding
services you would provide in your autonomous core. For instance, the
EVL core implements them as `evl_schedule()`, `evl_suspend_thread()`
and `evl_release_thread()`.

You may want to keep this in mind when going through the rest of this
document.

## Switching between execution stages {#stage-migration}

### Out-of-band switch  {#oob-switch}

Switching out-of-band is the operation by which a Linux task moves
under the control of the alternate scheduler the autonomous core adds
to the main kernel. From that point, the scheduling decisions are made
by this core regarding that task. There are two reasons a Linux task
may switch from in-band to out-of-band execution:

- either such transition is explicitly requested via a system call of
  the autonomous core, such as EVL's [evl_switch_oob()]({{% relref
  "core/user-api/thread/_index.md#evl_switch_oob" %}}).

- or the autonomous core has forced such transition, in response to a
  user request, such as issuing a system call which can only be
  handled from the out-of-band stage. The EVL core [ensures this]({{%
  relref "core/user-api/thread/_index.md" %}}).

Using Dovetail, a task which is executing on the in-band stage can
switch out-of-band following this sequence of actions:

1. this task calls [dovetail_leave_inband()]({{< relref
"#dovetail_leave_inband" >}}) from the in-band stage it runs on
(blue), which prepares for the transition, puts the caller to sleep
(`TASK_INTERRUPTIBLE`) then reschedules immediately. At this point,
the migrating task is in flight to the out-of-band stage (light
red). Meanwhile, `schedule()` resumes the next in-band task which
should run on the current CPU.

2. as the next in-band task context resumes, the scheduling tail code
checks for any task pending transition to out-of-band stage, _before_
the CPU is fully relinquished to the resuming in-band task. This check
is performed by the `inband_switch_tail()` call present in the main
scheduler. Such call has two purposes:

   * detect when a task is in flight to the out-of-band stage, so that
     we can notify the autonomous core for finalizing the migration
     process.

   * detect when a task is resuming on the out-of-band stage, which
     happens when the autonomous core switches context back to the
     current task, once the migration process is complete.

When switching out-of-band, case #1 is met, which triggers a call to
the [resume_oob_task()]({{< relref "#resume_oob_task" >}}) handler the
companion core should implement for completing the transition. This
would typically mean: unblock the migrating task from the standpoint
of its own scheduler, then reschedule. In the following flowchart,
`core_resume_task()` and `core_schedule()` stand for these two
operations, with each dotted link representing a context switch:

{{<mermaid align="left">}}
graph LR;
    S("start transition") --> A
    style S fill:#99ccff;
    A["dovetail_leave_inband()"] --> B["schedule()"]
    style A fill:#99ccff;
    style B fill:#99ccff;
    B -.-> C["inband_switch_tail()"]
    C --> D{task in flight?}
    D -->|Yes| E["resume_oob_task()"]
    style E fill:#99ccff;
    D -->|No| F{out-of-band?}
    E --> G["core_resume_task()"]
    style G fill:#99ccff;
    G --> H["core_schedule()"]
    style H fill:#ff950e;
    F -->|Yes| I(transition complete)
    style I fill:#ff950e;
    F -->|No| J(regular switch tail)
    style J fill:#99ccff;
    H -.-> C
{{< /mermaid >}}

At the end of this process, we should have observed a double context
switch, with the migrating task offloaded to the out-of-band
scheduler:

![Alt text](/images/oob_switch.png "Out-of-band switch")

{{% notice tip %}}
[evl_switch_oob()]({{% relref
"core/user-api/thread/_index.md#evl_switch_oob" %}}) implements the
switch to out-of-band context in the EVL core, with support from
`evl_release_thread()` and `evl_schedule()` for resuming and
rescheduling threads respectively.
{{% /notice %}}

### In-band switch {#inband-switch}

Switching in-band is the operation by which a Linux task moves under
the control of the main scheduler, coming from the out-of-band
execution stage. From that point, the scheduling decisions are made by
the main kernel regarding that task, and it may be subject to
preemption by in-band interrupts as well. In other words, _a task
switching in-band looses all guarantees regarding bounded response
time_; however, it regains access to the entire set of GPOS services
in the same move.

There are several reasons a Linux task which was running out-of-band
so far may have to switch in-band:

- it has requested it explicitly using a system call provided by the
  autonomous core such as EVL's [evl_switch_inband()]({{% relref
  "core/user-api/thread/_index.md#evl_switch_inband" %}}).

- the autonomous core has forced such transition for a task running in
  user-space:

  * in response to some request this task did, such as issuing a
    regular system call which can only be handled from the in-band
    stage. Typically, this behavior is enforced by the [EVL core]({{%
    relref "core/user-api/thread/_index.md" %}}).

  * because the task received a synchronous fault or exception, such
    as a memory access violation, FPU exception and so on. A demotion
    is required, because handling such events directly from the
    out-of-band stage would require a fair amount of code duplication,
    and most likely raise all sorts of funky conflicts between the
    out-of-band handlers and several in-band sub-systems. Besides,
    there is not much point in expecting real-time guarantees from a
    code that basically fixes up a situation caused by a
    dysfunctioning application in the first place.

  * a signal is pending for the task. Because the main kernel logic
    may require signals to be acknowledged by the recipient, we have
    to transition through the in-band stage to make sure the pending
    signal(s) will be delivered asap.

Using Dovetail, a task which executes on the out-of-band stage moves
in-band following this sequence of actions:

1. the execution of an in-band handler is scheduled from the context
   of the migrating task on the out-of-band stage. Once it runs, this
   handler should call `wake_up_process()` to unblock that task from
   the standpoint of the main kernel scheduler, since it is sleeping
   in `TASK_INTERRUPTIBLE` state there. Typically, the
   [_irq\_work_]({{% relref
   "dovetail/pipeline/pipeline_inject.md#irq-work" %}}) mechanism can
   be used for this, because:

   * as extended by the interrupt pipeline support, this interface can
     be used from the out-of-band stage.

   * the handler is guaranteed to run on the CPU from which the
     request was issued. Because the in-band work will wait until the
     out-of-band activity quiesces on that CPU, this in turn ensures
     all other operations we have to carry out from the
     out-of-band stage for preparing the migration are done before the
     task is woken up eventually.

2. the autonomous core blocks/suspends the migrating task,
   rescheduling immediately afterwards. For instance, the EVL core
   adds the `T_INBAND` block bit to the task's state for this purpose.

3. at some point later, the out-of-band context is exited by the
   current CPU when no more out-of-band work is left, causing the
   in-band kernel code to resume execution at the latest preemption
   point. The handler scheduled at step #1 eventually runs, waking up
   the migrating task from the standpoint of the main kernel. The
   `TASK_RUNNING` state is set for the task.

4. the migrating task resumes from the tail scheduling code of the
   alternate scheduler, where it suspended in step #2. Noticing the
   migration, the core calls [dovetail_resume_inband()]({{< relref
   "#dovetail_resume_inband" >}}) eventually, for finalizing the
   transition of the incoming task to the in-band stage.

In the following flowchart, `core_suspend_task()` and
`core_schedule()` stand for the operations described at step #2, with
each dotted link representing a context switch. The out-of-band idle
state represents the CPU transitioning from out-of-band (light red) to
in-band (blue) execution stage, as the core has no more out-of-band
task to schedule:

{{<mermaid align="left">}}
graph LR;
    S("start transition") --> A
    style S fill:#ff950e;
    A["irq_work(wakeup_req)"] --> B["core_suspend_task()"]
    style A fill:#ff950e;
    style B fill:#ff950e;
    B --> C["core_schedule()"]
    style C fill:#ff950e;
    C -.-> Y((OOB idle))
    Y -.-> D["wake_up_process()"]
    style D fill:#99ccff;
    D --> E["schedule()"]
    style E fill:#99ccff;
    E -.-> X["out-of-band switch tail"]
    style X fill:#99ccff;
    X --> G["dovetail_resume_inband()"]
    style G fill:#99ccff;
    G --> I("transition complete")
    style I fill:#99ccff;
{{< /mermaid >}}

At the end of this process, the task has transitioned from a running
state to a blocked state in the autonomous core, and conversely from
`TASK_INTERRUPTIBLE` to `TASK_RUNNING` in the main scheduler.

![Alt text](/images/inband_switch.png "In-band switch")

{{% notice tip %}}
[evl_switch_inband()]({{< relref
"core/user-api/thread/_index.md#evl_switch_inband" >}}) switches the caller to
in-band context in the EVL core, with support from
`evl_suspend_thread()` and `evl_schedule()` for suspending and
rescheduling threads respectively.  {{% /notice %}}

## Switching tasks out-of-band {#context-switching}

Dovetail allows an autonomous core embedded into the main kernel to
schedule a set of Linux tasks _out-of-band_ compared to the regular
_in-band_ kernel work, so that:

- the worse-case response time of such tasks only depends on the
  performance of a software core which has a limited complexity.

- the main kernel's logic does not have to cope with stringent bounded
  response time requirements, which otherwise tends to lower the
  throughput while making the design, implementation and maintenance
  significantly more complex.

The flowchart below represents the typical context switch sequence
an autonomous core should implement, from the context of the outgoing
_PREV_ task to the incoming _NEXT_ task:

{{<mermaid align="left">}}
graph LR;
    S(PREV) --> A["core_schedule()"]
    A --> B{in-band IRQ stage?}
    B -->|Yes| D["jump out-of-band"]
    D --> A
    B -->|No| C["pick NEXT"]
    style C fill:#ff950e;
    C --> P{PREV == NEXT?}
    style P fill:#ff950e;
    P -->|Yes| Q(no change)
    style Q fill:#ff950e;
    P -->|No| H["dovetail_context_switch()"]
    style H fill:#ff950e;
    H -.-> I(NEXT)
{{< /mermaid >}}

Those steps are:

1. `core_schedule()` is called over the _PREV_ context for picking the
_NEXT_ task to schedule, by priority order. If _PREV_ still has the
highest priority among all runnable tasks on the current CPU, the
sequence stops there. CAVEAT: the core _must make sure to perform
context switches from the out-of-band execution stage_, otherwise
weird things may happen down the road. [run_oob_call()]({{% relref
"dovetail/pipeline/stage_escalation.md" %}}) is a routine the
interrupt pipeline provides which may help there. See the
implementation of `evl_schedule()` in the EVL core for a typical
usage.

2. [dovetail_context_switch()]({{< relref "#dovetail_context_switch"
>}}) is called, switching the memory context as/if required, and the
CPU register file to _NEXT_'s, saving _PREV_'s in the same move.  If
_NEXT_ **is not** the [low priority placeholder task]({{< relref
"#altsched-theory" >}}) but _PREV_ is, we will be preempting the
in-band kernel: in this case, we must tell the kernel about such
preemption by passing _leave\_inband=true_ to
[dovetail_context_switch()]({{< relref "#dovetail_context_switch"
>}}).

3. _NEXT_ resumes from its latest switching point, which may be:

   * the switch tail code in `core_schedule()`, if _NEXT_ was running
     out-of-band prior to sleeping, in which case
     [dovetail_context_switch()]({{< relref "#dovetail_context_switch"
     >}}) returns _false_.

   * the switch tail code of `schedule()` if _NEXT_ is completing an
     [in-band switch]({{< relref "#inband-switch" >}}), in which case
     [dovetail_context_switch()]({{< relref "#dovetail_context_switch"
     >}}) returns _true_.

---

{{< proto dovetail_context_switch >}}
bool dovetail_context_switch(struct dovetail_altsched_context *prev, struct dovetail_altsched_context *next, bool leave_inband)
{{< /proto >}}

{{% argument prev %}}
The [alternate scheduling context block]({{< relref
"#dovetail_init_altsched" >}}) of the outgoing task.
{{% /argument %}}

{{% argument next %}}
The [alternate scheduling context block]({{< relref
"#dovetail_init_altsched" >}}) of the incoming task.
{{% /argument %}}

{{% argument leave_inband %}}
A boolean indicating whether we are leaving the in-band tasking mode,
which happens when _prev_ is the autonomous core's [low priority
placeholder task]({{< relref "#altsched-theory" >}}) standing for the
in-band kernel context as a whole.
{{% /argument %}}

This routine performs an out-of-band context switch. It must be called
with hard IRQs off. The arch-specific
[arch_dovetail_context_resume()]({{< relref
"#arch_dovetail_context_resume" >}}) handler is called by the resuming
task before leaving [dovetail_context_switch()]({{< relref
"#dovetail_context_switch" >}}). This _weak_ handler should be
overriden by a Dovetail port which requires arch-specific tweaks for
completing the reactivation of _next_. For instance, the arm64 port
performs the _fpsimd_ management from this handler.

[dovetail_context_switch()]({{< relref "#dovetail_context_switch"
>}}) returns a boolean value telling the caller
whether the current task just returned from a [transition from
out-of-band to in-band context]({{< relref "#inband-switch" >}}).

---

{{< proto dovetail_leave_inband >}}
int dovetail_leave_inband(void)
{{< /proto >}}

[dovetail_leave_inband()]({{< relref "#dovetail_leave_inband" >}})
should be called by your companion core in order to perform the
[out-of-band switch]({{< relref "#oob-switch" >}}) for the current
task.

On success, zero is returned, and the calling context is running on
the [out-of-band stage]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}). Otherwise,
-ERESTARTSYS is returned if a signal was pending at the time of the
call, in which case the transition could not take place.

> The usage is illustrated by the [implementation of
  evl_switch_oob()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/sched/core.c?h=evl/master)
  in the EVL core.

---

{{< proto dovetail_resume_inband >}}
void dovetail_resume_inband(void)
{{< /proto >}}

[dovetail_resume_inband()]({{< relref "#dovetail_resume_inband" >}})
should be called by your companion core as part of the [in-band switch
process]({{< relref "#inband-switch" >}}) for the current task,
_after_ the current task has resumed on the in-band execution stage
from the out-of-band suspension call in step 2 of the [in-band switch
process]({{< relref "#inband-switch" >}}). This *mandatory* call
finalizes the transition to this stage, by reconciling the current
task state with the internal state of the in-band scheduler.

> The usage is illustrated by the [implementation of
  evl_switch_inband()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/sched/core.c?h=evl/master)
  in the EVL core.

---

{{< proto resume_oob_task >}}
__weak void resume_oob_task(void)
{{< /proto >}}

This handler should be implemented by your companion core, in order to
complete step 2 of the [out-of-band switch process]({{< relref
"#oob-switch" >}}). Basically, this handler should lift the blocking
condition added to the task at step 2 of the [in-band switch
process]({{< relref "#inband-switch" >}}) which denotes in-band
execution, such as `T_INBAND` for the EVL core.

[resume_oob_task]({{< relref "#resume_oob_task" >}}) is called with
interrupts **disabled** in the CPU, out-of-band stage is stalled.

> An [implementation of
  resume_oob_task()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/sched/core.c?h=evl/master)
  is present in the EVL core.

## The event notifier {#event-notifier}

Once [dovetail_start_altsched()]({{< relref "#dovetail_start_altsched"
>}}) has been called for a regular in-band task, it may receive events
of interest with respect to running under the supervision of an
autonomous core. Those events are delivered by invoking handlers which
should by implemented by this core.

### Out-of-band exception handling {#oob-events}

If a processor exception is raised while the CPU is busy running a
task on the out-of-band stage (e.g. due to some invalid memory access,
bad instruction, FPU or alignment error etc.), the [task has to leave
such context]({{< relref "#inband-switch" >}}) before it may run the
in-band fault handler. Dovetail notifies the core about incoming
exceptions early from the low-level fault trampolines, but only when
some out-of-band code was running when the exception was taken. The
core may then fix up the current context, such as switching to the
in-band execution stage.

{{% notice tip %}}
Enabling debuggers to trace tasks running on the out-of-band stage
involves dealing with debug traps `ptrace()` may poke into the
debuggee's code for breakpointing.
{{% /notice %}}

The notification is delivered to the [handle_oob_trap()]({{< relref
"#handle_oob_trap" >}}) handler the core should override for receiving
those events (_\_\_weak_ binding). [handle_oob_trap()]({{< relref
"#handle_oob_trap" >}}) is passed the exception code as defined in
_arch/*/include/asm/dovetail.h_, and a pointer to the register frame
of the faulting context (_struct pt\_regs_).

---

{{< proto handle_oob_trap >}}
__weak void handle_oob_trap(unsigned int trapnr, struct pt_regs *regs)
{{< /proto >}}

This handler is called whenever a CPU trap is received by a
[Dovetail-enabled task]({{< relref "#dovetail_start_altsched" >}})
while running on the [out-of-band execution stage]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}). In such an
event, the caller must be switched to the in-band stage, in order to
safely perform the normal trap handling operations on return.  In
other words, Dovetail invokes this handler to ask your companion core
to switch back to a safe in-band context, before the in-band kernel
can actually handle such trap.

Obviously, the caller would lose the benefit of running on the
out-of-band stage, inducing latency in the process, but since it has
taken a CPU trap, it looks like things did not go as expected already
anyway. So the best option in this case is to switch the caller to
in-band mode, leaving the actual trap handling and any related fixup
to the in-band code once the transition is done.

{{% argument trapnr %}}

The trap code number. Such code depends on the CPU architecture:

- the documented Intel trap numbers are used for x86 (#GP, #DE, #OF etc.)
- other architectures may use a Dovetail-specific enumeration defined in
  `arch/*/include/asm/dovetail.h`.

{{% /argument %}}

{{% argument regs %}}
The register file at the time of the trap.
{{% /argument %}}

Interrupts are **disabled** in the CPU when this handler is called.

> An [implementation of
  handle_oob_trap()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/thread.c?h=evl/master)
  is present in the EVL core.

### System calls {#syscall-events}

The autonomous core is likely to introduce its own set of system calls
application tasks may invoke. From the standpoint of the in-band
kernel, this is a foreign set of calls, which can be distinguished
unambiguously from regular ones. If a task attached to the core issues
any system call, regardless of which of the kernel or the core should
handle it, the latter must be given the opportunity to:

- handle the request directly, possibly switching the caller to
  out-of-band context first if required.

- pass the request downward to the in-band system call path on the
  in-band stage, possibly switching the caller to in-band context if
  needed.

Dovetail intercepts system calls early in the kernel entry code,
delivering them to one of these handlers the core should override:

- the call is delivered to the [handle_oob_syscall()]({{< relref
"#handle_oob_syscall" >}}) handler if the system call number is not in
the valid range for the in-band kernel - i.e. it has to belong to the
core instead -, and the caller issued the request from the out-of-band
context. This is the fast path, when a task running out-of-band is
requesting a service the core provides.

- otherwise the slow path is taken, in which
[handle_pipelined_syscall()]({{< relref "#handle_pipelined_syscall"
>}}) is probed for handling the request from the appropriate execution
stage.  In this case, Dovetail performs the following actions:

{{<mermaid align="left">}}
graph LR;
    S("switch to out-of-band IRQ stage") --> A
    style S fill:#ff950e;
    A["handle_pipelined_syscall()"] --> B{returned zero?}
    B -->|Yes| C{on in-band IRQ stage?}
    B -->|No| R{on in-band IRQ stage?}
    R -->|Yes| U(branch to in-band user mode exit)
    R -->|No| V(done)
    C -->|Yes| Q(branch to in-band syscall handler)
    style Q fill:#99ccff;
    C -->|No| P[switch to in-band IRQ stage]
    style P fill:#ff950e;
    P --> A
{{< /mermaid >}}

In the flowchart above, [handle_pipelined_syscall()]({{< relref
"#handle_pipelined_syscall" >}}) should return zero if it wants
Dovetail to propagate an unhandled system call down the pipeline at
each of the two possible steps, non-zero if the request was
handled. Branching to the in-band user mode exit code ensures any
pending (in-band) signal is delivered to the current task and
rescheduling opportunities are taken when (in-band) kernel preemption
is enabled.

What makes a system call number out-of-range for the in-band kernel is
architecture-dependent. Some architectures may use the
most-significant bit in a syscall number as a differentiator
(i.e. regular if cleared, foreign if set), others may use a different
system call prefix to distinguish from the valid in-band call prefix.
See how `__EVL_SYSCALL_BIT` and `__ARM_NR_dovetail` are used for this
purpose in the [libevl]({{< relref "core/user-api/_index.md" >}})
implementation for arm64/x86 and ARM respectively.

Interrupts are always **enabled** in the CPU when any of these
handlers is called.

{{% notice note %}}
The core may need to switch the calling task to
the converse execution stage (i.e. in-band <-> out-of-band) either
from the [handle_oob_syscall()]({{< relref "#handle_oob_syscall" >}})
or [handle_pipelined_syscall()]({{< relref "#handle_pipelined_syscall"
>}}) handlers, this is fine. Dovetail would notice and continue
accordingly to the current stage on return of these handlers.  {{%
/notice %}}

---

{{< proto handle_oob_syscall >}}
__weak void handle_oob_syscall(struct pt_regs *regs)
{{< /proto >}}

This handler should implement the fast path for handling out-of-band
system calls.  It is called by Dovetail whenever a system call is
detected on entry of the common syscall path implemented by the
in-band kernel, which should be handled directly by the companion core
instead. Both of the following conditions must be met for this route
to be taken:

- the caller is a user task running on the [out-of-band execution
stage]({{< relref "dovetail/pipeline/_index.md#two-stage-pipeline"
>}}).

- the system call number is not in the range of the regular in-band
system call numbers.

A system call meeting those conditions denotes a request issued by an
application to a companion core which it can handle directly from its
native execution stage (i.e. [out-of-band]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}})). This handler
should be defined in the code of your companion core implementation
for receiving such system calls. Dovetail defines a dummy weak
implementation of it, which the implementation of the core would
supersede if defined (_\_\_weak_ binding).

The EVL core
[exhibits](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/syscall.c?h=evl/master)
a typical implementation of such a handler.

{{% argument regs %}}
The register file at the time of the system call, which contains the
call arguments passed by the application.
{{% /argument %}}

The handler should write the error code of the request to the
in-memory storage of the proper CPU register in _regs_, as defined by
the ABI convention for the CPU architecture.

[handle_oob_syscall]({{< relref "#handle_oob_syscall" >}}) is called
with interrupts enabled in the CPU, out-of-band stage is
unstalled. Whether the in-band stage accepts interrupts is undefined.

> An [implementation of
  handle_oob_syscall()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/syscall.c?h=evl/master)
  is present in the EVL core.

---

{{< proto handle_pipelined_syscall >}}
__weak void handle_pipelined_syscall(struct pt_regs *regs)
{{< /proto >}}

This handler is called by Dovetail whenever a system call is detected
on entry of the common syscall path implemented by the in-band kernel,
if the requirements for delivering it to the companion core via the
fast route implemented by [handle_oob_syscall]({{< relref
"#handle_oob_syscall" >}}) are not met, and any of the following
condition is true:

- the caller is a user task for which [alternate scheduling]({{<
relref "#dovetail_start_altsched" >}}) was enabled.

- the system call number is not in the range of the regular in-band
system call numbers, which means that a companion core might be able
to handle it (if not eventually, such a request would cause the caller
to receive -ENOSYS).

The handler should write the error code of the request to the
in-memory storage of the proper CPU register in _regs_, as defined by
the ABI convention for the CPU architecture.  In addition,
[handle_pipelined_syscall]({{< relref "#handle_pipelined_syscall" >}})
should return an integer status, which specifies the action Dovetail
should take on return:

- zero tells Dovetail to pass the system call to the regular in-band
  handler next. This status makes sense if the companion core did not
  handle the request, but did switch the calling context to in-band
  mode. This typically happens whenever the companion core wants to
  automatically demote the execution stage of the caller when it
  detects a regular in-band system call issued over the wrong
  (i.e. out-of-band) context, in which case it may [switch it
  in-band]({{< relref "#inband-switch" >}}) automatically for
  preserving the system integrity, before asking Dovetail to forward
  the request to the right (in-band) handler.

- a strictly positive status tells Dovetail to branch to the system
  call exit path immediately. This status makes sense if the companion
  core did handle the request, leaving the caller on the out-of-band
  execution stage.

- a negative status tells Dovetail to branch to the regular system
  call epilogue, _without_ passing the system call to the regular
  in-band handler though. This status makes sense if the companion
  core already handled the request, switching to in-band mode in the
  process. In such an event, the in-band kernel still wants to check
  for pending signals and rescheduling opportunity, which is the
  purpose of the epilogue code.

[handle_pipelined_syscall]({{< relref "#handle_pipelined_syscall" >}})
is called with interrupts enabled in the CPU, out-of-band stage is
unstalled. If current, the in-band stage accepts interrupts, otherwise
whether the in-band stage is stalled or not is undefined.

> An [implementation of
  handle_pipelined_syscall()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/syscall.c?h=evl/master)
  is present in the EVL core.

### In-band events {#inband-events}

The last set of notifications involves pure in-band events which the
autonomous core may need to know about, as they may affect its own
task management. Except for `INBAND_PROCESS_CLEANUP` which is called for
*any* exiting user-space task, all other notifications are only issued
for tasks bound to the core (which may involve kthreads).

The notification is delivered to the [handle_inband_event()]({{<
relref "#handle_inband_event" >}}) handler. The execution context of
this handler is always in-band. The out-of-band and in-band stages are
both [unstalled]({{< relref
"dovetail/pipeline/interrupt_protection.md" >}}) at the time of the
call. The notification handler receives the event type code, and a
single pointer argument which depends on the event type. The following
events are defined (see _include/linux/dovetail.h_):

- INBAND_TASK_SIGNAL(struct task_struct *target)

  	sent when @target is about to receive a signal. The core may
  decide to schedule a transition of the recipient to the in-band
  stage in order to have it handle that signal asap, which is required
  for keeping the kernel sane. This notification is always sent from
  the context of the issuer.

- INBAND_TASK_MIGRATION(struct dovetail_migration_data *p)

  	sent when p->task is about to move to CPU p->dest_cpu.

- INBAND_TASK_EXIT(struct task_struct *current)

  	sent from `do_exit()` before the current task has dropped the
  files and mappings it owns.

- INBAND_PROCESS_CLEANUP(struct mm_struct *mm)

  	sent before *mm* is entirely dropped, before the mappings are
  exited. Per-process resources which might still be maintained by the
  core could be released there, as all tasks sharing this memory
  context have exited. Unlike other events, this one is triggered for
  *every* user-space task which is being dismantled by the kernel,
  regardless of whether some of its threads were known from your
  autonomous core.

- INBAND_TASK_RETUSER(void)

	sent as a result of arming the return-to-user request via a
  call to [dovetail_request_ucall()]({{< relref
  "#dovetail_request_ucall" >}}). In other words, Dovetail fires this
  event because you asked for it by calling the latter service. The
  [INBAND_TASK_RETUSER]({{< relref "#inband-events" >}}) event handler
  in your companion core is allowed to switch the caller to the
  out-of-band stage before returning, ensuring the application code
  resumes in that context.

- INBAND_TASK_PTSTOP(void)

	sent when the current in-band task is about to sleep in
  ptrace_stop(), which is the place a task is supposed to wait until a
  debugger allows it to continue. A user-space task which receives
  SIGTRAP as a result of hitting a breakpoint, or SIGSTOP from the
  [ptrace(2)](http://man7.org/linux/man-pages/man2/ptrace.2.html)
  machinery parks itself by calling ptrace_stop(), until resumed by a
  [ptrace(2)](http://man7.org/linux/man-pages/man2/ptrace.2.html)
  request.

- INBAND_TASK_PTCONT(void)

	sent when the current in-band task is waking up from
  ptrace_stop() after the debugger allowed it to continue. The task
  may return to user-space afterwards, or go handling some pending
  signals.

- INBAND_TASK_PTSTEP(struct task_struct *target)

	sent by the
  [ptrace(2)](http://man7.org/linux/man-pages/man2/ptrace.2.html)
  implementation when it is about to (single-)step the @target
  task. Like PTSTOP and PTCONT, PTSTEP can be used to synchronize your
  companion core with the
  [ptrace(2)](http://man7.org/linux/man-pages/man2/ptrace.2.html)
  logic. For instance, EVL uses these events to provide support for
  [synchronous breakpoints]({{< relref
  "logbook/_index.md#announce-syncbp" >}}) when debugging applications
  over [gdb](https://www.gnu.org/software/gdb/).

---

{{< proto handle_inband_event >}}
__weak void handle_inband_event(enum inband_event_type event, void *data)
{{< /proto >}}

The handler which should be defined in the code of your companion core
implementation for receiving [in-band event]({{< relref
"#inband-events" >}}) notifications. Dovetail defines a dummy weak
implementation of it, which your implementation would supersede if
defined.

{{% argument event %}}
The [event code]({{< relref "#inband-events" >}}) (INBAND_TASK\_*)
as defined above.
{{% /argument %}}

{{% argument data %}}
An opaque pointer to some data further qualifying the event, which
actual type depends on the [event code]({{< relref "#inband-events" >}}).
{{% /argument %}}

The in-band stage is always current and accepts interrupts on entry to
this call.

> An [implementation of
  handle_inband_event()](https://git.evlproject.org/linux-evl.git/tree/kernel/evl/thread.c?h=evl/master)
  is present in the EVL core.

## Alternate task context {#dovetail-task-context}

Your autonomous core will need to keep its own set of per-task data,
starting with the [alternate scheduling context block]({{< relref
"#dovetail_init_altsched" >}}). To help in maintaining such
information on a per-task, per-architecture basis, Dovetail adds the
_oob\_state_ _member of type struct oob\_thread\_state_ to the
(stack-based) thread information block (aka _struct thread\_info_)
each kernel architecture port defines.

You may want to fill that structure reserved to out-of-band support
with any information your core may need for maintaining a task
context. By having this information defined in a file accessible from
the architecture-specific code by including
_\<dovetail/thread\_info.h\>_, the core-specific structure is
automatically added to _struct thread\_info_ as required. For
instance:

> arch/arm/include/dovetail/thread_info.h
```
struct oob_thread_state {
       /* Define your core-specific per-task data here. */
};
```

The core may then retrieve the address of the structure by calling
[dovetail_current_state()]({{< relref "#dovetail_current_state"
>}}). For instance, this is the definition the EVL core has for the
_oob\_thread\_state_ structure, storing a backpointer to its own task
control block, along with a core-specific preemption count for fast
stack-based access:

```
struct evl_thread;

struct oob_thread_state {
	struct evl_thread *thread;
	int preempt_count;
};
```

Which gives the following chain:

{{<mermaid align="left">}}
graph LR;
    T("struct thread_info") --> |includes| t
    t("struct oob_thread_state") -.-> |refers to| c
    c("struct evl_thread")
{{< /mermaid >}}

Note that we should not add all of the out-of-band state data directly
into the _oob\_thread\_state_ structure, because the latter is present
in **every** _task\_struct_ descriptor, although only a few tasks may
actually need it. Hence the backpointer to the _evl\_thread_ structure
which only consumes a few bytes, so leaving it unused (NULL) for all
other - in-band only - tasks is no big deal.

---

{{< proto dovetail_current_state >}}
struct oob_thread_state *dovetail_current_state(void)
{{< /proto >}}

This call retrieves the address of the out-of-band data structure for
the current task, which is always a valid pointer. The content of this
structure is zeroed when the task is created, and stays so until your
autonomous core initializes it.

## Extended memory context {#dovetail-mm-context}

Your autonomous core may also need to keep its own set of per-process
data.  To help in maintaining such information on a per-_mm_ basis,
Dovetail adds the `oob_state` member of type `struct oob_mm_state` to
the generic `struct mm_struct` descriptor . Because kernel threads can
only borrow memory contexts temporarily but do not actually own any,
this Dovetail extension is only available to EVL threads running in
user-space, **NOT** to threads created by [evl_run_kthread()]({{<
relref "core/kernel-api/kthread/_index.md#evl_run_kthread" >}}).

You may want to fill that structure reserved to out-of-band support
with any information your core may need for maintaining a per-_mm_
context. By having this information defined in a file accessible from
the architecture-specific code by including `\<dovetail/mm_info.h\>`,
the core-specific structure is automatically added to `struct
mm_struct` as required. For instance:

> arch/arm/include/dovetail/mm_info.h
```
struct oob_mm_state {
       /* Define your core-specific per-mm data here. */
};
```

The core may then retrieve the address of the structure for the
current task by calling [dovetail_mm_state()]({{< relref
"#dovetail_mm_state" >}}). The EVL core does not define any extended
memory context yet, but it could be used to maintain a per-process
file descriptor table for instance.

{{% notice tip %}}
Catching the [INBAND_PROCESS_CLEANUP]({{< relref "#inband-events" >}})
event would allow you to drop any resources attached to the extended
memory context information, since it is called when this data is no
longer in use by any thread of the exiting process.
{{% /notice %}}

---

{{< proto dovetail_mm_state >}}
struct oob_mm_state *dovetail_mm_state(void)
{{< /proto >}}

This call retrieves the address of the out-of-band data structure
within the _mm_ descriptor of the current user-space task. The content
of this structure is zeroed by the in-band kernel when it creates the
memory context, and stays so until your autonomous core initializes
it. If called from a kernel thread, NULL is returned instead.

> Definition of dovetail_mm_state()
```
static inline
struct oob_mm_state *dovetail_mm_state(void)
{
	if (current->flags & PF_KTHREAD)
		return NULL;

	return &current->mm->oob_state;
}
```

## Intercepting return to in-band user mode {#inband-ret-to-user}

In some specific cases, the implementation of your companion core may
need some thread running in-band to call back before it resumes
execution of the application code in user mode. For instance, you
might want to force that thread to switch back to the out-of-band
stage before it leaves the kernel and resumes user mode execution. For
this to happen, you would need that thread to jump back to the core at
that point, which would do the right thing from
there. [dovetail_request_ucall]({{< relref "#dovetail_request_ucall"
>}}) allows that.

---

{{< proto dovetail_request_ucall >}}
void dovetail_request_ucall(struct task_struct *target)
{{< /proto >}}

{{% argument target %}}
The task which should call back at the first opportunity. _target_
should have enabled the alternate scheduling support by a previous
call to [dovetail_start_altsched()]({{< relref
"#dovetail_start_altsched" >}}). If not, [dovetail_request_ucall]({{<
relref "#dovetail_request_ucall" >}}) has no effect.
{{% /argument %}}

Pend a request for _target_ to fire the [INBAND_TASK_RETUSER]({{<
relref "#inband-events" >}}) event at the first opportunity, which
happens whenever the task is about to resume execution in user mode
from the in-band stage.

If the [INBAND_TASK_RETUSER]({{< relref "#inband-events" >}}) event
handler in the companion core sends any signal to the current task, it
will be delivered before the application code resumes. The event
handler may pend a request to be called back again using
[dovetail_request_ucall()]({{< relref "#dovetail_request_ucall"
>}}).

---

{{<lastmodified>}}
