---
title: "Kernel thread"
weight: 5
---

### Out-of-band threads in kernel space

The EVL core can run common kernel threads on the out-of-band stage,
which can be used in [out-of-band capable drivers]({{< relref
"core/oob-drivers/_index.md" >}}) when ultra-low response time is
required.

---

{{< proto evl_run_kthread >}}
int evl_run_kthread(struct evl_kthread *kthread, void (*threadfn)(struct evl_kthread *kthread), int priority, int clone_flags, const char *fmt, ...)
{{< /proto >}}

[evl_run_kthread()]({{% relref "#evl_run_kthread" %}}) is a
macro-definition which spawns an EVL kernel thread, which is the EVL
equivalent of its in-band kernel counterpart named `kthread_run()`.

The new thread may be pinned on any of the out-of-band capable CPUs
(See the `evl.oob_cpus` kernel parameter). If you need to spawn a
kernel thread on a particular CPU, you may want to use
[evl_run_kthread_on_cpu()]({{% relref "#evl_run_kthread_on_cpu" %}})
instead.

{{% argument kthread %}}
A kernel thread descriptor where the core will maintain the per-thread
context information. This memory area must remain valid as long as the
associated kernel thread runs.
{{% /argument %}}

{{% argument threadfn %}}
The routine to run in the new thread context, which is passed
_kthread_ as its sole argument.
{{% /argument %}}

{{% argument priority %}}
The priority of the new thread, which is assumed to refer to the
[SCHED_FIFO]({{% relref
"core/user-api/scheduling/_index.md#SCHED_FIFO" %}}) class.
{{% /argument %}}

{{% argument clone_flags %}}
A set of creation flags for the new kernel thread, defining its
[visibility]({{< relref "core/user-api/_index.md#element-visibility"
>}}):
- `EVL_CLONE_PUBLIC` denotes a public thread which is represented
   by a device file in the [/dev/evl]({{< relref
   "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy, which
   makes it visible to application processes for sharing.
  
- `EVL_CLONE_PRIVATE` denotes a thread which is private to the
   kernel. No device file appears for it in the
   [/dev/evl]({{< relref "core/user-api/_index.md#evl-fs-hierarchy" >}})
   file hierarchy.
{{% /argument %}}

{{% argument fmt %}}
A `ksprintf()`-like format string to generate the thread name. Unlike
[evl_attach_thread()]({{% relref
"core/user-api/thread/_index.md#evl_attach_thread" %}}) from the user API,
[evl_run_kthread()]({{% relref "#evl_run_kthread" %}}) does not look
for any shorthand defined by the [naming convention]({{< relref
"core/user-api/_index.md#element-naming-convention" >}}) for
application threads. Thread visibility can be set exclusively by using
the _clone_flags_.
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_run_kthread()]({{% relref "#evl_run_kthread" %}}) returns zero on
success, or a negated error code otherwise:

- -EEXIST	The generated name is conflicting with an existing thread
		name.

- -EINVAL	Either _clone_flags_ or _priority_ are wrong.

- -ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

- -ENOMEM	Not enough memory available. Buckle up.

---

{{< proto evl_run_kthread_on_cpu >}}
int evl_run_kthread_on_cpu(struct evl_kthread *kthread, int cpu, void (*threadfn)(struct evl_kthread *kthread), int priority, int clone_flags, const char *fmt, ...)
{{< /proto >}}

As its name suggests, [evl_run_kthread_on_cpu()]({{% relref
"#evl_run_kthread_on_cpu" %}}) is a variant of [evl_run_kthread()]({{%
relref "#evl_run_kthread" %}}) which lets you pick a particular CPU
for pinning the new kernel thread.

{{% argument cpu %}}
The CPU number the new thread should be pinned to, among the
out-of-band capable ones (See the `evl.oob_cpus` kernel parameter).
{{% /argument %}}

[evl_run_kthread_on_cpu()]({{% relref "#evl_run_kthread_on_cpu" %}})
returns zero on success, or a negated error code. The set of error
conditions for [evl_run_kthread()]({{% relref "#evl_run_kthread" %}})
apply to [evl_run_kthread_on_cpu()]({{% relref
"#evl_run_kthread_on_cpu" %}}), plus:

- -EINVAL	_cpu_ is not a valid, out-of-band capable CPU.

---

{{< proto evl_stop_kthread >}}
void evl_stop_kthread(struct evl_kthread *kthread)
{{< /proto >}}

This call requests the EVL _kthread_ to exit at the first
opportunity. It may be called from any stage, but only from a kernel
thread context, regular in-band or EVL.

This is an advisory method for stopping EVL kernel threads, which
requires _kthread_ to call [evl_kthread_should_stop()]({{%
relref "#evl_kthread_should_stop" %}}) as part of its regular work
loop, exiting when such test returns _true_.  In other words,
[evl_stop_kthread()]({{% relref "#evl_stop_kthread" %}}) raises the
condition which [evl_kthread_should_stop()]({{% relref
"#evl_kthread_should_stop" %}}) returns to its caller.

[evl_stop_kthread()]({{% relref "#evl_stop_kthread" %}}) first
unblocks _kthread_ from any blocking call, then waits for _kthread_ to
actually exit before returning to the caller. Therefore, an EVL kernel
thread which receives a request for termination while sleeping on some
EVL call unblocks with -EINTR as a result.

{{% notice tip %}}
There is no way to forcibly terminate kernel threads since this
would potentially leave the kernel system in a broken, unstable state.
Both the requestor and the subject kernel thread must cooperate for the
later to follow an orderly process for exiting. The
in-band equivalent is achieved with
[kthread_stop()](https://www.kernel.org/doc/html/latest/driver-api/basics.html#c.kthread_stop),
[kthread_should_stop()](https://www.kernel.org/doc/html/latest/driver-api/basics.html#c.kthread_should_stop).
{{% /notice %}}

{{% argument kthread %}}
The EVL kernel thread to send a stop request to. If _kthread_ represents the calling
context (i.e. self-termination), the call does not return and the caller is
exited immediately. Otherwise, the stop request is left pending until _kthread_
eventually calls [evl_kthread_should_stop()]({{% relref "#evl_kthread_should_stop"
%}}), at which point it should act upon this event by exiting.
{{% /argument %}}

---

{{< proto evl_kthread_should_stop >}}
bool evl_kthread_should_stop(void)
{{< /proto >}}

This call is paired with [evl_stop_kthread()]({{% relref
"#evl_stop_kthread" %}}). It should be called by any EVL kernel thread
which intends to accept termination requests from other threads.

Whenever [evl_kthread_should_stop()]({{% relref
"#evl_kthread_should_stop" %}}) returns _true_, the caller should plan
for exiting as soon as possible, typically by returning from its entry
routine (_threadfn_ argument to [evl_run_kthread()]({{% relref
"#evl_run_kthread" %}})). Otherwise, it may continue.

A typical usage pattern is as follows:

```

#include <evl/thread.h>
#include <evl/flag.h>

static DEFINE_EVL_FLAG(some_flag);

void some_kthread(struct evl_kthread *kthread)
{
	int ret;

	for (;;) {
		if (evl_kthread_should_stop())
			break;

		/* wait for the next processing signal */
		ret = evl_wait_flag(&some_flag);
		if (ret == -EINTR)
		    	break;

		/* do some useful stuff */
	}

	/* about to leave, do some cleanup */
}
```

---

![Alt text](/images/wip.png "To be continued")

---

#### evl_set_kthread_priority(struct evl_kthread *thread, int priority)

---

#### ktime_t evl_delay(ktime_t timeout, enum evl_tmode timeout_mode, struct evl_clock *clock)

---

#### int evl_sleep_until(ktime_t timeout)

---

#### int evl_sleep(ktime_t delay)

---

#### int evl_set_period(struct evl_clock *clock, ktime_t idate, ktime_t period)

---

#### int evl_wait_period(unsigned long *overruns_r)

---

{{<lastmodified>}}
