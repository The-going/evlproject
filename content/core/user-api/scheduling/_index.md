---
menutitle: "Scheduling"
weight: 52
---

### The out-of-band scheduler

EVL defines five scheduling policies for running out-of-band threads.
These policies are hierarchized: every time the core needs to pick the
next eligible thread to run on the current CPU, it queries each policy
module for a runnable thread in the following order:

1. [SCHED_FIFO]({{< relref "#SCHED_FIFO" >}}), which is the common
   first-in, first-out real-time policy, also dealing with the
   [SCHED_RR]({{< relref "#SCHED_RR" >}}) round-robin policy
   internally.

2. [SCHED_TP]({{< relref "#SCHED_TP" >}}), which enforces temporal
   partitioning of multiple sets of threads in a way which prevents
   those sets from overlapping time-wise on the CPU which runs such
   policy.

3. [SCHED_QUOTA]({{< relref "#SCHED_QUOTA" >}}), which enforces a
   limitation on the CPU consumption of threads over a fixed period of
   time.

4. [SCHED_WEAK]({{< relref "#SCHED_WEAK" >}}), which is a *non
   real-time* policy allowing its members to run in-band most of the
   time, while retaining the ability to request EVL services.

5. [SCHED_IDLE]({{< relref "#SCHED_IDLE" >}}), which is the fallback
   option the EVL core considers only when other policies have no
   runnable task on the CPU.

The SCHED_QUOTA and SCHED_TP policies are optionally supported by the
core, make sure to enable `CONFIG_EVL_SCHED_QUOTA` or
`CONFIG_EVL_SCHED_TP` respectively in the kernel configuration if you
need them.

{{% notice tip %}}
Before a thread can be assigned to any EVL class, its has to attach
itself to the core by a call to [evl_attach_self]({{% relref
"core/user-api/thread/_index.md#evl_attach_self" %}}).
{{% /notice %}}

### Scheduler services {#sched-services}

{{< proto evl_set_schedattr >}}
int evl_set_schedattr(int efd, const struct evl_sched_attrs *attrs)
{{< /proto >}}

This call changes the scheduling attributes for the thread referred to
by _efd_ in the EVL core.

{{% argument efd %}}
A file descriptor referring to the target thread, as returned by
[evl_attach_self()]({{% relref
"core/user-api/thread/_index.md#evl_attach_self" %}}),
[evl_get_self()]({{% relref
"core/user-api/thread/_index.md#evl_get_self" %}}), or opening a
thread element device in _/dev/evl/thread_ using
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html).
{{% /argument %}}

{{% argument attrs %}}
A structure defining the new set of attributes, which depends
on the scheduling policy mentioned in
`attrs->sched_policy`. EVL currently implement the following
policies:
{{% /argument %}}

- [SCHED_FIFO]({{% relref "#SCHED_FIFO" %}}), which is the common
  first-in, first-out real-time policy.

- [SCHED_RR]({{% relref "#SCHED_RR" %}}), defining a real-time,
round-robin policy in which each member of the class is allotted an
individual time quantum before the CPU is given to the next thread.

- [SCHED_TP]({{% relref "#SCHED_TP" %}}), which enforces temporal
  partitioning of multiple sets of threads based on a cycle-based
  scheduling.

- [SCHED_QUOTA]({{% relref "#SCHED_QUOTA" %}}), which enforces a
  limitation on the CPU consumption of threads over a fixed period of
  time, known as the global quota period. Threads undergoing this
  policy are pooled in groups, with each group being given a share of
  the period.

- [SCHED_WEAK]({{% relref "#SCHED_WEAK" %}}), which is a *non
  real-time* policy allowing its members to run in-band most of the
  time, while retaining the ability to request EVL services, at the
  expense of briefly switching to the out-of-band execution stage on
  demand.

[evl_set_schedattr()]({{% relref "#evl_set_schedattr" %}}) returns zero on success, otherwise a negated
error code is returned:

-EBADF		_efd_ is not a valid thread descriptor.

-EINVAL		Some of the parameters in _attrs_ are wrong. Check
		`attrs->sched_policy`, and the policy-specific
		information may EVL expect for more.

-ESTALE		_efd_ refers to a stale thread, see these [notes]({{< relref
		"core/user-api/thread/_index.md#evl_detach_self" >}}).

```
#include <evl/thread.h>

int change_self_schedparams(void)
{
	struct evl_sched_attrs attrs;

	attrs.sched_policy = SCHED_FIFO;
	attrs.sched_priority = 90;
	return evl_set_schedattr(evl_get_self(), &attrs);
}
```

[evl_set_schedattr()]({{% relref "#evl_set_schedattr" %}}) immediately
changes the scheduling attributes the EVL core uses for the target
thread when it runs in out-of-band context. Later on, the next time
such thread transitions from out-of-band to in-band context, the main
kernel will apply an extrapolated version of those changes to its own
scheduler as well.

The extrapolation of the out-of-band scheduling attributes passed to
[evl_set_schedattr()]({{% relref "#evl_set_schedattr" %}}) to the
in-band ones applied by the mainline kernel works as follows:

| out-of-band policy    | in-band policy |
| ------------------    | -------------- |
| SCHED_FIFO, prio      |    SCHED_FIFO, prio  |
| SCHED_RR, prio        |    SCHED_FIFO, prio  |
| SCHED_TP, prio        |    SCHED_FIFO, prio  |
| SCHED_QUOTA, prio     |    SCHED_FIFO, prio  |
| SCHED_WEAK, prio > 0  |    SCHED_FIFO, prio  |
| SCHED_WEAK, prio == 0 |    SCHED_OTHER |

{{% notice warning %}}
Calling
[pthread_setschedparam(3)](http://man7.org/linux/man-pages/man3/pthread_setschedparam.3.html)
from the C library does not affect the scheduling attributes of an EVL
thread. It only affects the scheduling parameters of such thread from
the standpoint of the main kernel. Because the C library may cache the
current scheduling attributes for the in-band context of a thread -
_glibc_ does so typically - the cached value may not reflect the
actual scheduling attributes of the thread after this call.
{{% /notice %}}

---

{{< proto evl_get_schedattr >}}
int evl_get_schedattr(int efd, struct evl_sched_attrs *attrs)
{{< /proto >}}

This is the call for retrieving the current scheduling attributes of
the thread referred to by _efd_.

{{% argument efd %}}
A file descriptor referring to the target thread, as returned by
[evl_attach_self()]({{< relref
"core/user-api/thread/_index.md#evl_attach_self" >}}),
[evl_get_self()]({{< relref
"core/user-api/thread/_index.md#evl_get_self" >}}), or opening a
thread element device in _/dev/evl/thread_ using `open(2)`.
{{% /argument %}}

{{% argument attrs %}}
A pointer to a structure where the EVL core should write back the
scheduling attributes.
{{% /argument %}}

```
#include <evl/thread.h>

int retrieve_self_schedparams(void)
{
	struct evl_sched_attrs attrs;

	return evl_get_schedattr(evl_get_self(), &attrs);
}
```

The value returned in `attrs.sched_priority` is the base priority
level of the thread within its scheduling class, which does NOT
reflect any priority inheritance/ceiling boost that might be ongoing.

[evl_get_schedattr()]({{% relref "#evl_get_schedattr" %}}) returns
zero on success, otherwise a negated error code is returned:

-EBADF		_efd_ is not a valid thread descriptor.

-ESTALE		_efd_ refers to a stale thread, see these [notes]({{< relref
		"core/user-api/thread/_index.md#evl_detach_self" >}}).

---

{{< proto evl_control_sched >}}
int evl_control_sched(int policy, const union evl_sched_ctlparam *param, union evl_sched_ctlinfo *info, int cpu)
{{< /proto >}}

---

### Scheduling policies

#### SCHED_FIFO {#SCHED_FIFO}

The first-in, first-out policy, fixed priority, preemptive scheduling
policy. If you really need a refresher about this one, you can still
have a look at this inspirational piece of post-modern poetry for
[background info](
https://pubs.opengroup.org/onlinepubs/009695399/functions/xsh_chap02_08.html).
Or you can just go for the short version: with SCHED_FIFO, the
scheduler always picks the runnable thread with the highest priority
which spent the longest time waiting for the CPU to be available.

EVL provides 99 fixed priority levels starting a 1, which maps 1:1 to
the main kernel's SCHED_FIFO implementation as well.

Switching a thread to FIFO scheduling is achieved by calling
[evl_set_schedattr()]({{% relref "#evl_set_schedattr" %}}). The
`evl_sched_attrs` attribute structure should be filled in as follows:

```
	struct evl_sched_attrs attrs;

	attrs.sched_policy = SCHED_FIFO;
	attrs.sched_priority = <priority>; /* [1-99] */

```
---

#### SCHED_RR {#SCHED_RR}

The round-robin policy is based on SCHED_FIFO
internally. Additionally, it limits the execution time of its members
to a given _timeslice_, moving a thread which fully consumed its
current timeslice to the tail of the scheduling queue for its priority
level. This is designed as a simple way to prevent threads from
over-consuming the CPU within their own priority level.

Unlike the main kernel which defines a global timeslice value for all
members of the SCHED_RR class, EVL defines a per-thread quantum
instead. Since EVL is tickless, this quantum may be any valid
duration, and may differ among threads from the same priority group.

Switching a thread to round-robin scheduling is achieved by calling
[evl_set_schedattr()]({{% relref "#evl_set_schedattr" %}}). The
`evl_sched_attrs` attribute structure should be filled in as follows:

```
	struct evl_sched_attrs attrs;

	attrs.sched_policy = SCHED_RR;
	attrs.sched_priority = <priority>; /* [1-99] */
	attrs.sched_rr_quantum = (struct timespec){
			       .tv_sec = <seconds>,
			       .tv_nsec = <nanoseconds>,
			       };

```
---

#### SCHED_QUOTA {#SCHED_QUOTA}

The quota-based policy enforces a limitation on the CPU consumption of
threads over a fixed period of time, known as the global quota
period. Threads undergoing this policy are pooled in groups, with each
group being given a share of the period (expressed as a
percentage). The rules of the SCHED_FIFO policy apply among all
threads regardless of their group.

For instance, say that we have five distinct thread groups, each of
which is given a runtime credit which represents a portion of the
global period: 35%, 25%, 15%, 10% and finally 5% of a global period
set to one second. The first group would be allotted 350 milliseconds
over a second, the second group would get 250 milliseconds from the
same period and so on.

Every time a thread undergoing the SCHED_QUOTA policy is given the
CPU, the time it consumes is charged to the group it belongs
to. Whenever the group as a whole reaches the alloted time credit, all
its members stall until the next period starts, at which point the
runtime credit of every group is replenished for the next round of
execution, resuming all its members in the process.

You may attach as many threads as you need to a single group, and the
number of threads may vary among groups. The alloted runtime quota for
a group is decreased by the execution time of every thread in that
group. Therefore, a group with no thread does not consume its quota.

![Alt text](/images/quota_scheduling.png "Quota-based scheduling")

Switching a thread to quota-based scheduling is achieved by calling
[evl_set_schedattr()]({{% relref "#evl_set_schedattr" %}}). The
`evl_sched_attrs` attribute structure should be filled in as follows:

```
	struct evl_sched_attrs attrs;

	attrs.sched_policy = SCHED_QUOTA;
	attrs.sched_priority = <priority>; /* [1-99] */
	attrs.sched_quota_group = <grpid>; /* Quota group id. */
```

[Creating]({{< relref "#create-quota-group" >}}), [modifying]({{<
relref "#modify-quota-group" >}}) and [removing]({{< relref
"#remove-quota-group" >}}) thread groups is achieved by calling
[evl_control_sched()]({{% relref "#evl_control_sched" %}}).

##### Runtime credit and peak quota {#sched-peak-quota}

Each thread group is given its full quota every time the global period
starts, according to the [configuration set]({{< relref
"#modify-quota-group" >}}) for this group. If the group did not
consume its quota entirely by the end of the current period, the
remaining credit is added to the group's quota for the next period, up
to a limit defined as the _peak quota_. If the accumulated credit
would cause the quota to exceed the peak value, the extra time is
spread over multiple subsequent periods until the credit is fully
consumed.

##### Creating a quota group {#create-quota-group}

EVL supports up to 1024 distinct thread groups for quota-based
scheduling system-wide. Each thread group is assigned to a specific
CPU by the application code which creates it, among the set of CPUs
EVL runs out-of-band activity on (see the `evl.oobcpus` kernel
parameter).

A thread group is represented by a unique integer returned by the core
upon creation, aka the _group identifier_.

Creating a new thread group is achieved by calling
[evl_sched_contol()]({{% relref "#evl_control_sched" %}}). Some
information including the new group identifier is returned in the
ancillary `evl_sched_ctlinfo` structure passed to the request. The
`evl_sched_ctlparams` control structure should be filled in as
follows:

```
	union evl_sched_ctlparam param;
	union evl_sched_ctlinfo info;
	int ret;

	param.quota.op = evl_quota_add;
	ret = evl_control_sched(SCHED_QUOTA, &param, &info, <cpu-number>);
```

On success, the following information is received from the core
regarding the new group:

- info.quota.tgid contains the new group identifier.

- info.quota.quota_percent is the current percentage of the global
  period allotted to the new group. At creation time, this value is
  set to 100%. You may want to [change it]({{< relref
  "#modify-quota-group" >}}) to reflect the final value.

- info.quota.quota_peak_percent reflects the current [_peak
  percentage_]({{< relref "#sched-peak-quota" >}}) of the global
  period allotted to the new group, which is also set to 100% at
  creation time. This means that by default, a new group might double
  its quota value by accumulating runtime credit, consuming up to 100%
  of the CPU time during the next period. Likewise, you may want to
  [change it]({{< relref "#modify-quota-group" >}}) to reflect the
  final value.

- info.quota.quota_sum is the sum of the quota values of all groups
  assigned to the CPU specified in the [evl_control_sched()]({{%
  relref "#evl_control_sched" %}}) request. This gives the overall CPU
  business as far as SCHED_QUOTA is concerned. This sum should not
  exceed 100% for a CPU in a properly configured system.

##### Modifying a quota group {#modify-quota-group}

##### Removing a quota group {#remove-quota-group}

---

#### SCHED_TP {#SCHED_TP}

This policy enforces a so-called _temporal partitioning_, which is a
way to schedule thread activities on a given CPU so that they cannot
overlap time-wise. To this end, the policy defines a global, _major_
time frame of fixed duration repeating cyclically, which is divided
into smaller _minor_ frames or _time windows_ also of fixed durations,
but not necessarily equal.  Therefore, a minor frame is bounded by its
time offset from the beginning of the major frame, and its own
duration. The sum of all minor frame durations defines the duration of
the global time frame.

Each minor frame contributes to the run-time allotted within the
global time frame to a group of thread called a _partition_.  The
maximum number of partitions in the system is defined by
`CONFIG_EVL_TP_NR_PART` in the kernel configuration. Each thread
undergoing this policy is attached to one of such _partitions_.

When a minor frame elapses on a CPU, the threads attached to its
partition are immediately suspended, and threads from the partition
assigned to the next minor frame may be picked for scheduling. When
the last minor frame elapses, the process repeats from the minor frame
leading the major time frame.

You may assign as many threads as you need to a single partition, and
the number of threads may vary among partitions. A partition with no
thread simply runs no SCHED_TP thread until the next minor frame
assigned a non-empty partition starts.

![Alt text](/images/temporal_partitioning.png "Temporal partitioning")

{{% notice tip %}}
Valid partition numbers range from 0 to `CONFIG_EVL_TP_NR_PART -
1`. The special partition number `EVL_TP_IDLE` can be used when
configuring the scheduler to designate the _idle_ partition when
assigning it to a minor frame, creating a time hole in the schedule
which will not run any SCHED_TP thread until this minor frame elapses.
{{% /notice %}}

Switching a thread to temporal partitioning is achieved by calling
[evl_set_schedattr()]({{% relref "#evl_set_schedattr" %}}). The
`evl_sched_attrs` attribute structure should be filled in as follows:

```
	struct evl_sched_attrs attrs;

	attrs.sched_policy = SCHED_TP;
	attrs.sched_priority = <priority>; /* [1-99] */
	attrs.sched_tp_partition = <ptid>; /* Partition id. */
```

_ptid_ should be a valid partition identifier between 0 and
`CONFIG_EVL_TP_NR_PART` - 1.

Within a minor time frame, threads undergo the SCHED_FIFO policy
according to their fixed _priority_ value.

##### Setting the temporal partitioning information for a CPU

Temporal partitioning is defined on a per-CPU basis, you have to
configure each CPU involved in this policy independently, enumerating
the minor frames which compose the major one as follows:

```
	union evl_sched_ctlparam param;
	int ret;

	param.tp.op = evl_tp_install;
	param.tp.nr_windows = <number_of_minor_frames>;
	param.tp.windows[0].offset = <offset_from_start_of_major_frame>;
	param.tp.windows[0].duration = <duration_of_minor_frame>;
	param.tp.windows[0].ptid = <assigned_partition_id>;
	...
	param.tp.windows[<number_of_minor_frames> - 1].offset = <offset_from_start_of_major_frame>;
	param.tp.windows[<number_of_minor_frames> - 1].duration = <duration_of_minor_frame>;
	param.tp.windows[<number_of_minor_frames> - 1].ptid = <assigned_partition_id>;

	ret = evl_control_sched(SCHED_TP, &param, NULL, <cpu-number>);
```
If the special value `EVL_TP_IDLE` is assigned to a minor time frame
(_ptid_), no thread undergoing the SCHED_TP policy will run until such
frame elapses. This is a way to create a _time hole_ within the major
time frame.

This operation implicitly overrides the previous TP settings for the
CPU, leaving the TP scheduling in stopped state. Generally speaking,
once a TP schedule is installed for a CPU, it is left in standby mode
until explicitly started.

##### Starting the TP scheduling on a CPU

The following request enables the TP scheduler for the given CPU; you
have to issue this request for activating a TP schedule:

```
	union evl_sched_ctlparam param;
	int ret;

	param.tp.op = evl_tp_start;
	ret = evl_control_sched(SCHED_TP, &param, NULL, <cpu-number>);
```

##### Stopping the TP scheduling on a CPU

You can disable the scheduling of threads undergoing the SCHED_TP
policy on a CPU by the following call:

```
	union evl_sched_ctlparam param;
	int ret;

	param.tp.op = evl_tp_stop;
	ret = evl_control_sched(SCHED_TP, &param, NULL, <cpu-number>);
```

Upon success, the target CPU won't be considered for running SCHED_TP
threads, until the converse request `evl_tp_start` is received for
that CPU.

##### Retrieving the temporal partitioning information from a CPU

The following request fetches the current scheduling plan for a given
CPU:

```
	union evl_sched_ctlparam param;
	struct {
		union evl_sched_ctlinfo info;
		struct __sched_tp_window windows[<max_number_of_minor_frames>];
	} result;
	int ret;

	param.tp.op = evl_tp_get;
	param.tp.nr_windows = <max_number_of_minor_frames>;
	ret = evl_control_sched(SCHED_TP, &param, &result.info, <cpu-number>);
```

`param.nr_windows[]` specifies the maximum number of minor frames the
EVL core should dump into the `result.info.nr_windows[]` output
variable array. This value is automatically capped to the actual
number of minor frames defined for the target CPU at the time of the
call.

The minor frames active for the target CPU are readable from the
`result.windows[]` array, up to `param.nr_windows[]` . The number of
valid elements in this array is given by `result.info.nr_windows[]`.

##### Removing the temporal partitioning information from a CPU

This is the converse operation to `evl_tp_install`, removing temporal
partitioning support for the target CPU, releasing all related
resources. This request implicitly stops the TP scheduling on such CPU
prior to uninstalling.

---

#### SCHED_WEAK {#SCHED_WEAK}

You may want to run some POSIX threads in-band most of the time,
except when they need to call some EVL services
occasionally. Occasionally here means either non-repeatedly, or at any
rate _not_ from a high frequency loop.

Members of this class are picked second to last in the hierarchy of
EVL scheduling classes, right before the sole member of the
[SCHED_IDLE]({{< relref "#SCHED_IDLE" >}}) class which stands for the
in-band execution stage of the kernel. The priority such threads have
in the in-band context are preserved by this policy when they
(normally briefly) run on the out-of-band execution stage. For this
reason, SCHED_WEAK threads have a fixed priority ranging from 0 to 99
included, which maps to in-band SCHED_OTHER (0), SCHED_FIFO and
SCHED_RR (1-99) priority ranges.

A thread scheduled in the SCHED_WEAK class may invoke any EVL service,
including blocking ones for waiting for out-of-band notifications
(e.g. depleting a semaphore), which will certainly switch it to the
out-of-band execution stage. Before returning from the EVL system
call, the thread will be automatically switched back to the in-band
execution stage by the core. This means that _each and every EVL
system call_ issued by a thread assigned to the SCHED_WEAK class is
going to trigger _two execution stage switches_ back and forth, which
is definitely _costly_. So make sure not to use this feature in any
high frequency loop.

{{% notice warning %}}
In a specific case the EVL core will keep the thread running on the
out-of-band execution stage though: whenever this thread has returned
from a successful call to [evl_lock_mutex()]({{% relref
"core/user-api/mutex/_index.md#evl_lock_mutex" %}}), holding an EVL
mutex. This ensures no in-band activity on the same CPU can
preempt the lock owner, which would certainly lead to a priority
inversion would that lock be contended later on by another EVL
thread. The lock owner is eventually switched back to in-band mode by
the core as a result of [releasing]({{% relref
"core/user-api/mutex/_index.md#evl_unlock_mutex" %}}) the last EVL mutex it
was holding.
{{% /notice %}}

There are only a few legitimate use cases for assigning an EVL thread
to the SCHED_WEAK scheduling class:

- as part of some initialization, cleanup or any non real-time phase
  of your application, a thread needs to synchronize with another EVL
  thread which belongs to a real-time class like [SCHED_FIFO]({{<
  relref "#SCHED_FIFO" >}}).

- some in-band thread which purpose is to handle _fairly exceptional_
  events needs to be notified by a real-time thread to do so.

In all other cases where an in-band thread might need to be driven by
out-of-band events which may occur at a moderate or higher rate, using
a message-based mechanism such as a [cross-buffer]({{% relref
"core/user-api/xbuf/_index.md" %}}) is the best way to go.

Also note that some EVL services may be called by regular POSIX
threads. Typically, a plain POSIX thread may [post an EVL
semaphore]({{% relref "core/user-api/semaphore/_index.md" %}}) for
signaling an out-of-band EVL thread pending on it. You can find the
allowed calling contexts for each EVL service from `libevl`
[there]({{% relref "core/user-api/function_index/_index.md" %}}).

Switching a thread to SCHED_WEAK is achieved by calling
[evl_set_schedattr()]({{% relref "#evl_set_schedattr" %}}). The
`evl_sched_attrs` attribute structure should be filled in as follows:

```
	struct evl_sched_attrs attrs;

	attrs.sched_policy = SCHED_WEAK;
	attrs.sched_priority = <priority>; /* [0-99] */

```
---

#### SCHED_IDLE {#SCHED_IDLE}

The idle class has a single task on each CPU: the [low priority
placeholder task]({{< relref
"dovetail/altsched.md#altsched-theory" >}}).

SCHED_IDLE has the lowest priority among policies, its sole task is
picked for scheduling only when other policies have no runnable task
on the CPU. A task member of the SCHED_IDLE class cannot block, it is
always runnable.

This is an internal class, which is neither available to user
applications nor drivers.

---

{{<lastmodified>}}
