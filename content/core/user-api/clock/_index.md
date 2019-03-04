---
menuTitle: "Clocks"
title: "Clock element"
weight: 25
---

The target platform can provide particular clock chips and/or clock
source drivers in addition to the architecture-specific ones. For
instance, some device on a PCI bus could implement a timer chip which
the application wants to use for timing its threads, in addition to
the architected timer found on ARM64 and some ARM-based SoCs. In this
case, we would need a specific _clock driver_, binding the timer
hardware to the EVL core.

EVL's clock element ensures that all clock drivers present the same
interface to applications in user-space. In addition, the clock
element can export individual software timers to applications which
comes in handy for running periodic loops or waiting for oneshot
events on a specific time base.

{{% notice note %}}
EVL abstracts clock event devices and clock sources
(timekeeping hardware) into a single _clock element_.
{{% /notice %}}

#### int evl_read_clock(int efd, struct timespec *tp)

---

#### int evl_set_clock(int efd, const struct timespec *tp)

---

#### int evl_get_clock_resolution(int efd, struct timespec *tp)

---

#### int evl_adjust_clock(int efd, struct timex *tx)

---

#### int evl_sleep(int efd, const struct timespec *timeout, struct timespec *remain)

---

#### evl_udelay(unsigned int usecs)
