---
menuTitle: "Clock device"
weight: 30
---

![Alt text](/images/wip.png "To be continued")

The target platform can provide particular clock chips and/or clock
source drivers in addition to the architecture-specific ones. For
instance, some device on a PCI bus could implement a timer chip which
the application wants to use for timing its threads, in addition to
the architected timer found on ARM64 and some ARM-based SoCs. In this
case, we would need a specific _clock driver_, binding the timer
hardware to the EVL core.

The EVL clock device interface gives you a framework for integrating
such hardware into the real-time core, which in turns enables
applications to [use it]({{% relref "core/user-api/clock/_index.md"
%}}).

{{% notice note %}}
EVL abstracts clock event devices and clock sources
(timekeeping hardware) into a single _clock element_.
{{% /notice %}}

---

{{< proto evl_init_clock >}}
int evl_init_clock(struct evl_clock *clock, const struct cpumask *affinity)
{{< /proto >}}

Initializes a new clock which may be used to drive [EVL core
timers]({{< relref "core/kernel-api/timer/_index.md" >}}).

{{% argument clock %}}
A descriptor describing the clock properties, which the core is also
going to use for maintaining some runtime information. See below.
{{% /argument %}}

{{% argument affinity %}}
The set of CPUs we may expect the backing clock device to tick on,
usually denoting a per-CPU device. As a special case, passing a NULL
affinity mask means that we have to deal with a global device which
does _not_ issue per-CPU events, in which case outstanding timers
based on this clock will be maintained into a single global queue
instead of per-CPU timer queues. At least one out-of-band CPU
mentioned in `evl_oob_cpus` should be present in the clock affinity
mask.
{{% /argument %}}

The clock descriptor should be filled in by the caller with the
following information on entry to [evl_init_clock()]({{< relref
"#evl_init_clock" >}}):

- **.name**: name of the clock. If the clock has public visibility,
  the file representing this clock in the [/dev/evl hierarchy]({{<
  relref "core/user-api/_index.md#evl-fs-hierarchy" >}}) is named
  after it.

- **.resolution**: the resolution of the clock in nanoseconds.

- **.flags**: a set of creation flags for the new clock, defining its
[visibility]({{< relref "core/user-api/_index.md#element-visibility"
>}}):
	- `EVL_CLONE_PUBLIC` denotes a public clock which is represented
	   by a device file in the [/dev/evl]({{< relref
	   "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy, which
	   makes it visible to application processes for sharing.
  
	- `EVL_CLONE_PRIVATE` denotes a clock which is private to the
	   kernel. No device file appears for it in the
	   [/dev/evl]({{< relref "core/user-api/_index.md#evl-fs-hierarchy" >}})
	   file hierarchy.

- **.gravity**: the clock gravity value, which is a set of static
  		adjustment values to account for the basic latency of
  		the target system for responding to timer events, as
  		described in [this document]({{< relref
  		"core/runtime-settings#calibrate-core-timer" >}}).

- **.ops**: a set of operation handlers which should be implemented by
    the caller:

	- ktime_t (*read)(struct evl_clock *clock)

	- u64 (*read_cycles)(struct evl_clock *clock)

	- int (*set)(struct evl_clock *clock, ktime_t date)

	- void (*program_local_shot)(struct evl_clock *clock)

	- void (*program_remote_shot)(struct evl_clock *clock, struct evl_rq *rq)

	- int (*set_gravity)(struct evl_clock *clock, const struct evl_clock_gravity *p)

	- void (*reset_gravity)(struct evl_clock *clock)

	- void (*adjust)(struct evl_clock *clock)

> A typical example: the EVL core monotonic clock

```
struct evl_clock evl_mono_clock = {
	.name = EVL_CLOCK_MONOTONIC_DEV,  /* i.e. "monotonic" */
	.resolution = 1,  /* nanosecond. */
	.flags = EVL_CLONE_PUBLIC,
	.ops = {
		.read = read_mono_clock,
		.read_cycles = read_mono_clock_cycles,
		.program_local_shot = evl_program_proxy_tick,
#ifdef CONFIG_SMP
		.program_remote_shot = evl_send_timer_ipi,
#endif
		.set_gravity = set_coreclk_gravity,
		.reset_gravity = reset_coreclk_gravity,
	},
};
```

[evl_init_clock()]({{% relref "#evl_init_clock" %}}) returns zero on
success, or a negated error code otherwise:

- -EEXIST	The generated name is conflicting with an existing clock
		name.

- -EINVAL	Either _clock->flags_ or _affinity_ are wrong.

- -ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

- -ENOMEM	Not enough memory available.

---

#### evl_init_slave_clock(struct evl_clock *clock, struct evl_clock *master)

---

#### int evl_register_clock(struct evl_clock *clock, const struct cpumask *affinity)

---

#### void evl_unregister_clock(struct evl_clock *clock)

---

#### void evl_announce_tick(struct evl_clock *clock)

---

#### void evl_adjust_timers(struct evl_clock *clock, ktime_t delta)

---

#### ktime_t evl_read_clock(struct evl_clock *clock)

---

#### ktime_t evl_ktime_monotonic(void)

---

#### u64 evl_read_clock_cycles(struct evl_clock *clock)

---

#### int evl_set_clock_time(struct evl_clock *clock, const struct timespec *ts)

---

#### void evl_set_clock_resolution(struct evl_clock *clock, ktime_t resolution)

---

#### ktime_t evl_get_clock_resolution(struct evl_clock *clock)

---

#### int evl_set_clock_gravity(struct evl_clock *clock, const struct evl_clock_gravity *gravity)

---

#### struct evl_clock *evl_get_clock_by_fd(int efd)

---

#### void evl_put_clock(struct evl_clock *clock)

---

{{<lastmodified>}}
