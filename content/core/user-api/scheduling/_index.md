---
title: "Scheduling policies"
weight: 52
---

EVL defines four scheduling policies, which are prioritized: this
means that every time the core needs to pick the next eligible thread
to run on the current CPU, it queries each policy module for a
runnable thread in the following order:

1. [SCHED_FIFO]({{< relref "#SCHED_FIFO" >}}), which is the common
   first-in, first-out real-time policy, also dealing with the
   [SCHED_RR]({{< relref "#SCHED_RR" >}}) round-robin policy
   internally.

2. [SCHED_QUOTA]({{< relref "#SCHED_QUOTA" >}}), which enforces a
   limitation on the CPU consumption of threads over a fixed period of
   time.

3. [SCHED_WEAK]({{< relref "#SCHED_WEAK" >}}), which is a *non
   real-time* policy allowing its members to run in-band most of the
   time, while retaining the ability to request EVL services.

4. [SCHED_IDLE]({{< relref "#SCHED_IDLE" >}}), which is the fallback
   option the EVL core considers only when other policies have no
   runnable task on the CPU.

{{% notice tip %}}
The SCHED_QUOTA policy is optionally supported by the core, make sure
to enable `CONFIG_EVL_SCHED_QUOTA` in the kernel configuration if
you need it.
{{% /notice %}}

---

### SCHED_FIFO {#SCHED_FIFO}

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
[`evl_set_schedattr()`]({{% relref
"core/user-api/thread/_index.md#evl_set_schedattr" %}}). The
`evl_sched_attrs` attribute structure should be filled in as follows:

```
	struct evl_sched_attrs attrs;

	attrs.sched_policy = SCHED_FIFO;
	attrs.sched_priority = <priority>; /* [1-99] */

```
---

### SCHED_RR {#SCHED_RR}

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
[`evl_set_schedattr()`]({{% relref
"core/user-api/thread/_index.md#evl_set_schedattr" %}}). The
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

### SCHED_QUOTA {#SCHED_QUOTA}

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
number of threads may vary between groups. The alloted runtime quota
for a group is decreased by the execution time of every thread in that
group. Therefore, a group with no thread does not consume its quota.

![Alt text](/images/quota_scheduling.png "Quota-based scheduling")

Switching a thread to quota-based scheduling is achieved by calling
[`evl_set_schedattr()`]({{% relref
"core/user-api/thread/_index.md#evl_set_schedattr" %}}). The
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
[`evl_sched_control()`]({{% relref
"core/user-api/utils/_index.md#evl_sched_control" %}}).

#### Runtime credit and peak quota {#sched-peak-quota}

Each thread group is given its full quota every time the global period
starts, according to the [configuration set]({{< relref
"#modify-quota-group" >}}) for this group. If the group did not
consume its quota entirely by the end of the current period, the
remaining credit is added to the group's quota for the next period, up
to a limit defined as the _peak quota_. If the accumulated credit
would cause the quota to exceed the peak value, the extra time is
spread over multiple subsequent periods until the credit is fully
consumed.

#### Creating a quota group {#create-quota-group}

EVL supports up to 1024 distinct thread groups for quota-based
scheduling system-wide. Each thread group is assigned to a specific
CPU by the application code which creates it, among the set of CPUs
EVL runs out-of-band activity on (see the `evl.oobcpus` kernel
parameter).

A thread group is represented by a unique integer returned by the core
upon creation, aka the _group identifier_.

Creating a new thread group is achieved by calling
[`evl_sched_contol()`]({{% relref
"core/user-api/utils/_index.md#evl_sched_control" %}}). Some
information including the new group identifier is returned in the
ancillary `evl_sched_ctlinfo` structure passed to the request. The
`evl_sched_ctlparams` control structure should be filled in as
follows:

```
	struct evl_sched_ctlparam param;
	struct evl_sched_ctlinfo info;

	param.quota.op = evl_quota_add;
	ret = evl_sched_control(SCHED_QUOTA, &param, &info, <cpu-number>);
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
  assigned to the CPU specified in the `evl_sched_control()`
  request. This gives the overall CPU business as far as SCHED_QUOTA
  is concerned. This sum should not exceed 100% for a CPU in a
  properly configured system.

#### Modifying a quota group {#modify-quota-group}

#### Removing a quota group {#remove-quota-group}

---

### SCHED_WEAK {#SCHED_WEAK}

---

### SCHED_IDLE {#SCHED_IDLE}

The idle class has a single task on each CPU: the [low priority
placeholder task]({{< relref
"dovetail/altsched/_index.md#altsched-theory" >}}).

SCHED_IDLE has the lowest priority among policies, its sole task is
picked for scheduling only when other policies have no runnable task
on the CPU. A task member of the SCHED_IDLE class cannot block, it is
always runnable
