---
title: "Alternate scheduling"
date: 2018-06-27T19:01:42+02:00
weight: 15
pre: "&rsaquo; "
---

Dovetail promotes the idea that a *dual kernel* system should keep the
functional overlap between the kernel and the autonomous core
minimal. To this end, a task from such core should be merely seen as a
regular task with additional scheduling capabilities guaranteeing very
low and bounded response times. To support such idea, Dovetail enables
kthreads and regular user tasks to run alternatively in the
out-of-band execution context introduced by the interrupt pipeline
(aka *oob* stage), or the common in-band kernel context for GPOS
operations (aka *in-band* stage).

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
  call to `dovetail_init_altsched()`. For instance, the EVL core does
  so as part of the attachment process of a thread to the autonomous
  core when [evl_attach_self()]({{% relref
  "core/user-api/thread/_index.md#evl_attach_self" %}}) is called by
  the application.

- once the alternate scheduling feature is initialized, the task
  should enable it by a call to `dovetail_start_altsched()`. From this
  point, that task:

  * can switch between the in-band and out-of-band execution stages
    freely.

  * can emit out-of-band system calls the autonomous core can handle.

  * is tracked by Dovetail's notification system, so that in-band and
    out-of-band events which may involve such task are dispatched to
    the core.

  * can be involved in Dovetail-based context switches triggered by
    the autonomous core, independently from the scheduling operations
    carried out by the main kernel.

    Conversely, `dovetail_stop_altsched()` disables the notifications
    for a task, which is likely to be detached from the autonomous core
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
  state]({{% relref "dovetail/pipeline/optimistic.md#virtual-i-flag"
  %}}) allows it.

![Alt text](/images/altsched.png "Alternate scheduling")

---

{{< proto dovetail_init_altsched >}}
void dovetail_init_altsched(struct dovetail_altsched_context *p)
{{< /proto >}}

This call initializes the alternate scheduling context for the current
task; this should be done once, before the task calls
`dovetail_start_altsched()`.

{{% argument p %}}
The alternate scheduling context is kept in a per-task structure of
type _dovetail_altsched_context_ which should be maintained by the
autonomous core. This can be done as part of the [per-task context
management]({{< relref "#dovetail-task-context" >}}) facility
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
_out-of-band interrupt stage_.

- In the first case, the task is not controlled by the main kernel
  scheduler anymore, but runs under the supervision of the autonomous
  core. This is obtained by performing an [out-of-band switch]({{<
  relref "#oob-switch" >}}) for a task.

- In the other case, the current underlying context (task or whatever
  else) may or may not be controlled by the main kernel, but the
  interrupt pipeline machinery has switched the CPU to out-of-band
  mode, which means that only interrupts bearing the IRQF_OOB flag are
  delivered. Typically, [`run_oob_call()`]({{% relref
  "dovetail/pipeline/usage/stage_escalation.md" %}}) is a service
  provided by the interrupt pipeline which executes a function call
  over this context, without requiring the calling task to be
  scheduled by the autonomous core.

You will also find references to pseudo-routines called
`core_schedule()`, `core_suspend_task()` or `core_resume_task()` in
various places. Don't look for them into the code, they don't actually
exist: those routines are mere placeholders for the corresponding
services you would provide in your autonomous core. For instance, the
EVL core implements them as `evl_schedule()`, `evl_suspend_thread()`
and `evl_release_thread()`.

You may want to keep this in mind when going through the rest of this
document.

## Migrating between execution stages {#stage-migration}

### Out-of-band switch  {#oob-switch}

Switching out-of-band is the operation by which a Linux task moves
under the control of the alternate scheduler brought in the main
kernel by the autonomous core. From that point, the scheduling
decisions are made by this core regarding that task.

There are two reasons a Linux task may switch out-of-band:

- either it has requested it explicitly using a system call provided
  by the autonomous core such as EVL's [evl_switch_oob()]({{% relref
  "core/user-api/thread/_index.md#evl_switch_oob" %}}).

- or the autonomous core has forced such transition, in response to
  some request this task did, such as issuing a system call which can
  only be handled from the out-of-band stage. This behavior is
  enforced by the [EVL core]({{% relref
  "core/user-api/thread/_index.md#thread-element" %}}) too.

Using Dovetail, a task which executes on the in-band stage moves
out-of-band following this sequence of actions:

1. this task calls `dovetail_leave_inband()`, which prepares for the
transition, puts the caller to sleep (`TASK_INTERRUPTIBLE`) then
reschedules immediately. At this point, the migrating task is in
flight to the out-of-band stage. `schedule()` resumes the next
in-band task which should run on the current CPU.

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
the `resume_oob_task()` handler the autonomous core should implement
for completing the transition. This would typically mean: unblock the
migrating task from the standpoint of its own scheduler, then
reschedule. In the following flowchart, `core_resume_task()` and
`core_schedule()` stand for these two operations, with each dotted
link representing a context switch:

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
`evl_switch_oob()` implements the switch to out-of-band context in the
EVL core, with support from `evl_release_thread()` and
`evl_schedule()` for resuming and rescheduling threads respectively.
{{% /notice %}}

### In-band switch {#inband-switch}

Switching in-band is the operation by which a Linux task moves under
the control of the main scheduler, coming from the out-of-band
execution stage. From that point, the scheduling decisions are made by
the main kernel regarding that task.

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
    relref "core/user-api/thread/_index.md#thread-element" %}}).

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
   "dovetail/pipeline/porting/irqflow.md#irq-work" %}}) mechanism can
   be used for this, because:

   * as extended by the interrupt pipeline support, this interface can
     be used from the out-of-band stage.

   * the handler is guaranteed to run on the CPU from which the
     request was issued. Because the in-band work will wait until the
     out-of-band activity quiesces on that CPU, this in turn ensures
     that all other operations we have to carry out from the
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
   migration, the core calls `dovetail_resume_inband()` eventually,
   for finalizing the transition of the incoming task to the in-band
   stage.

In the following flowchart, `core_suspend_task()` and
`core_schedule()` stand for the operations described at step #2, with
each dotted link representing a context switch. The out-of-band idle
state represents the CPU transitioning from out-of-band to in-band
execution stage, as the core has no more out-of-band task to schedule:

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
`evl_switch_inband()` implements the switch to in-band context in the
EVL core, with support from `evl_suspend_thread()` and
`evl_schedule()` for suspending and rescheduling threads respectively.
{{% /notice %}}

## Dovetail context switching {#context-switching}

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
    P -->|No| E{PREV == idle?}
    style E fill:#ff950e;
    E -->|Yes| F["dovetail_resume_oob()"]
    style F fill:#ff950e;
    E -->|No| H["dovetail_context_switch()"]
    style H fill:#ff950e;
    F --> H
    H -.-> I(NEXT)
{{< /mermaid >}}

Those steps are:

1. `core_schedule()` is called over the _PREV_ context for picking the
_NEXT_ task to schedule, by priority order. If _PREV_ still has the
highest priority among all runnable tasks on the current CPU, the
sequence stops there. CAVEAT: the core _must make sure to perform
context switches from the out-of-band interrupt stage_, otherwise
weird things may happen down the road. [`run_oob_call()`]({{% relref
"dovetail/pipeline/usage/stage_escalation.md" %}}) is a routine the
interrupt pipeline provides which may help there. See the
implementation of `evl_schedule()` in the EVL core for a typical
usage.

2. If _NEXT_ **is not** the [low priority placeholder task]({{< relref
"#altsched-theory" >}}) but _PREV_ is, we will be preempting the
in-band kernel: in this case, we must tell the kernel about such
preemption by a call to `dovetail_resume_oob()`. This routine saves
all context deemed necessary to restore it later on when the core
switches back to the in-band execution stage. Otherwise, if _NEXT_ is
the placeholder task, there is no out-of-band task waiting for the
CPU, and we will branch back to the last preemption point that led
to calling `dovetail_resume_oob()` during the converse transition.

3. `dovetail_context_switch()` is called, switching the memory context
as/if required, and the CPU register file to _NEXT_'s, saving _PREV_'s
in the same move.

4. _NEXT_ resumes from its latest switching point, which may be:

   * the switch tail code in `core_schedule()`, if _NEXT_ was running
     out-of-band prior to sleeping.

   * the switch tail code of `schedule()` if _NEXT_ is completing an
     [out-of-band switch]({{< relref "#oob-switch" >}}).

---

{{< proto dovetail_context_switch >}}
void dovetail_context_switch(struct dovetail_altsched_context *prev, struct dovetail_altsched_context *next)
{{< /proto >}}

{{% argument prev %}}
The [alternate scheduling context block]({{< relref
"#dovetail_init_altsched" >}}) of the outgoing task.
{{% /argument %}}

{{% argument next %}}
The [alternate scheduling context block]({{< relref
"#dovetail_init_altsched" >}}) of the incoming task.
{{% /argument %}}

This routine performs an out-of-band context switch. It must be called
with hard IRQs off. The arch-specific `arch_dovetail_context_resume()`
handler is called by the resuming task before leaving
`dovetail_context_switch()`. This _weak_ handler should be overriden
by a Dovetail port which requires arch-specific tweaks for completing
the reactivation of _next_. For instance, the arm64 port performs the
_fpsimd_ management from this handler.

---

{{< proto dovetail_resume_oob >}}
void dovetail_resume_oob(struct dovetail_altsched_context *outgoing)
{{< /proto >}}

{{% argument outgoing %}}
The [alternate scheduling context block]({{< relref
"#dovetail_init_altsched" >}}) of the outgoing context, which should
be the [low priority placeholder task's]({{< relref "#altsched-theory"
>}}).
{{% /argument %}}

This routine informs the kernel that we are leaving the in-band
context, right before resuming an out-of-band task. This allows the
main kernel logic to save all the information that will be required to
restore the in-band execution stage later on when the converse switch
happens.  The only call site of `dovetail_resume_oob()` should be in
the `core_schedule()` implementation.

---

## The event notifier {#event-notifier}

Once `dovetail_start_altsched()` has been called for a Linux task, it
may receive events of interest with respect to running under the
supervision of an autonomous core. Those events are delivered by
invoking _\_\_weak_ handlers which should by overriden by this
core.

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

The notification is delivered to the `handle_oob_trap()` handler the
core should override for receiving those events (_\_\_weak_
binding). `handle_oob_trap()` is passed the exception code as defined
in _arch/*/include/asm/dovetail.h_, and a pointer to the register
frame of the faulting context (_struct pt\_regs_).

Interrupts are **disabled** in the CPU when this handler is called.

### System calls {#syscall-events}

The autonomous core is likely to introduce its own set of system calls
application tasks may invoke. From the standpoint of the main kernel,
this is a foreign set of calls, which can be distinguished
unambiguously from regular ones.

If a task attached to the core issues any system call, regardless of
which of the kernel or the core should handle it, the latter must be
given the opportunity to:

- handle the request directly, possibly switching the caller to
  out-of-band context first if required.

- pass the request downward to the normal system call path on the
  in-band stage, possibly switching the caller to in-band context if
  needed.

Dovetail intercepts system calls early in the kernel entry code,
delivering them to one of these handlers the core should override:

- the call is delivered to the `handle_oob_syscall()` handler if the
system call number is not in the valid range for the in-band kernel -
i.e. it has to belong to the core instead -, and the caller issued the
request from the out-of-band context. This is the fast path, when a
task running out-of-band is requesting a service the core provides.

- `handle_pipelined_syscall()` is a slower path, in which this handler
is probed for the appropriate execution stage in order to handle the
request.  In this case, the calling logic is as follows:

{{<mermaid align="left">}}
graph LR;
    S("from out-of-band IRQ stage") --> A
    style S fill:#ff950e;
    A["handle_pipelined_syscall()"] --> B{returned zero?}
    B -->|Yes| C{on in-band stage?}
    B -->|No| R(done)
    C -->|Yes| Q(handle as regular kernel syscall)
    style Q fill:#99ccff;
    C -->|No| P[switch to in-band IRQ stage]
    style P fill:#ff950e;
    P --> A
{{< /mermaid >}}

In the flowchart above, `handle_pipelined_syscall()` should return
zero if it wants Dovetail to propagate the unhandled system call down
the pipeline, non-zero if the request was handled. The propagation is
carried out:

- first by re-running `handle_pipelined_syscall()` in the in-band
  context coming from the out-of-band one.

- then by passing the syscall to the regular in-band system call
handler in the assembly entry code eventually, if it remained
unhandled by `handle_pipelined_syscall()`.

Interrupts are **enabled** in the CPU when any of these handlers is
called.

{{% notice note %}}
The core may need to switch the calling task to the converse execution
stage (i.e. in-band <-> out-of-band) either from the
`handle_oob_syscall()` or `handle_pipelined_syscall()` handlers, this
is fine. Dovetail would notice and reconcile its logic according to
the current stage on return of these handlers.
{{% /notice %}}

### In-band events {#inband-events}

The last set of notifications involves pure in-band events which the
autonomous core may need to know about, as they may affect its own
task management. Except for `INBAND_PROCESS_CLEANUP` which is called for
*any* exiting user-space task, all other notifications are only issued
for tasks bound to the core (which may involve kthreads).

The notification is delivered to the `handle_inband_event()` handler.
Interrupts are **enabled** in the CPU when this handler is called.

The notification handler is given the event type code, and a single
pointer argument which relates to the event type.

The following events are defined (see _include/linux/dovetail.h_):

- INBAND_TASK_SCHEDULE(struct task_struct *next)

  sent in preparation of a context switch, right before the memory
  context is switched to *next*.

- INBAND_TASK_SIGNAL(struct task_struct *target)

  sent when *target* is about to receive a signal. The core may decide
  to schedule a transition of the recipient to the in-band stage in
  order to have it handle that signal asap, which is required for
  keeping the kernel sane. This notification is always sent from the
  context of the issuer.

- INBAND_TASK_MIGRATION(struct dovetail_migration_data *p)

  sent when p->task is about to move to CPU p->dest_cpu.

- INBAND_TASK_EXIT(struct task_struct *current)

  sent from `do_exit()` before the current task has dropped the files
  and mappings it owns.

- INBAND_PROCESS_CLEANUP(struct mm_struct *mm)

  sent before *mm* is entirely dropped, before the mappings are
  exited. Per-process resources which might still be maintained by the
  core could be released there, as all tasks sharing this memory
  context have exited. Unlike other events, this one is triggered for
  *every* user-space task which is being dismantled by the kernel,
  regardless of whether some of its threads were known from your
  autonomous core.

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
[`dovetail_current_state()`]({{< relref "#dovetail_current_state"
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

---

## Extended memory context {#dovetail-mm-context}

Your autonomous core may also need to keep its own set of per-process
data.  To help in maintaining such information on a per-_mm_ basis,
Dovetail adds the _oob\_state_ member of type _struct oob\_mm\_state_
to the generic _mm_ descriptor structure (aka _struct mm\_struct_).

You may want to fill that structure reserved to out-of-band support
with any information your core may need for maintaining a per-_mm_
context. By having this information defined in a file accessible from
the architecture-specific code by including _\<dovetail/mm\_info.h\>_,
the core-specific structure is automatically added to _struct
mm\_struct_ as required. For instance:

> arch/arm/include/dovetail/mm_info.h
```
struct oob_mm_state {
       /* Define your core-specific per-mm data here. */
};

```

The core may then retrieve the address of the structure for the
current task by calling [`dovetail_mm_state()`]({{< relref
"#dovetail_mm_state" >}}). The EVL core does not define any extended
memory context yet, but it could be used to maintain a per-process
file descriptor table for instance.

{{% notice tip %}}
Catching the [`INBAND_PROCESS_CLEANUP`]({{< relref "#inband-events" >}})
event would allow you to drop any resources attached to the extended
memory context information, since it is called when this data is no
longer in use by any thread of the exiting process.
{{% /notice %}}

---

{{< proto dovetail_mm_state >}}
struct oob_mm_state *dovetail_mm_state(void)
{{< /proto >}}

This call retrieves the address of the out-of-band data structure
within the _mm_ descriptor of the current task, which is always a
valid pointer. The content of this structure is zeroed when the memory
context is created, and stays so until your autonomous core
initializes it.

---
