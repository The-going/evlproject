---
menuTitle: "Clock devices"
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

EVL's clock device interface gives you a framework for integrating
such hardware into the real-time core, which in turns enables
applications to [use it]({{% relref "core/user-api/clock/_index.md"
%}}).

{{% notice note %}}
EVL abstracts clock event devices and clock sources
(timekeeping hardware) into a single _clock element_.
{{% /notice %}}

#### int evl_init_clock(struct evl_clock *clock,	const struct cpumask *affinity)

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
