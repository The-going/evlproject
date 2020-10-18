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

![Alt text](/images/wip.png "To be continued")

---

#### void evl_cancel_kthread(struct evl_kthread *kthread)

---

#### int evl_kthread_should_stop(void)

---

#### evl_set_kthread_priority(struct evl_kthread *thread, int priority)

---

#### int evl_sleep_until(ktime_t timeout)

---

#### int evl_sleep(ktime_t delay)

---

#### int evl_set_thread_period(struct evl_clock *clock, ktime_t idate, ktime_t period)

---

#### int evl_wait_thread_period(unsigned long *overruns_r)

---

{{<lastmodified>}}
