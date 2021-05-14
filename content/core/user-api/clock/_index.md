---
menuTitle: "Clocks"
weight: 25
---

### Clock element {#clock-element}

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

Some [built-in clocks]({{< relref "#builtin-clocks" >}}) are
pre-defined by the EVL core for reading the monotonic and wallclock
time of the system.
  
{{% notice note %}}
EVL abstracts clock event devices and clock sources
(timekeeping hardware) into a single _clock element_.
{{% /notice %}}

### Clock services {#clock-services}

{{< proto evl_read_clock >}}
int evl_read_clock(int clockfd, struct timespec *tp)
{{< /proto >}}

This call is the EVL equivalent of the POSIX
[clock_gettime(3)](http://man7.org/linux/man-pages/man3/clock_gettime.3.html)
service, for reading the time from a specified EVL clock.  It reads
the current time maintained by the EVL clock, which is returned as
counts of seconds and microseconds since the clock's _epoch_.

{{% argument clockfd %}}
The clock file descriptor, which can be:

- any valid file descriptor received as a result of opening a clock
  device in _/dev/evl/clock_.

- the identifier of a [built-in EVL clock]({{< relref
  "#builtin-clocks" >}}), such as `EVL_CLOCK_MONOTONIC` or
  `EVL_CLOCK_REALTIME`.
{{% /argument %}}

{{% argument tp %}}
A pointer to a _timespec_ structure which should receive the time
stamp.
{{% /argument %}}

`evl_read_clock()` writes the timestamp to _tp_ then returns zero on
success, otherwise:

- -EBADF if _clockfd_ is neither a built-in clock identifier or a valid
   file descriptor.

- -ENOTTY if _clockfd_ does not refer to an EVL clock device.

- -EFAULT if _tp_ points to invalid memory.

---

{{< proto evl_set_clock >}}
int evl_set_clock(int clockfd, const struct timespec *tp)
{{< /proto >}}

This call is the EVL equivalent of the POSIX
[clock_settime(3)](http://man7.org/linux/man-pages/man3/clock_settime.3.html)
service, for setting the time of a specified EVL clock.

{{% argument clockfd %}}
The clock file descriptor, which can be:

- any valid file descriptor received as a result of opening a clock
  device in _/dev/evl/clock_.

- the identifier of a [built-in EVL clock]({{< relref
  "#builtin-clocks" >}}), such as `EVL_CLOCK_MONOTONIC` or
  `EVL_CLOCK_REALTIME`.
{{% /argument %}}

{{% argument tp %}}
A pointer to a _timespec_ structure which should receive the time
stamp.
{{% /argument %}}

`evl_set_clock()` returns zero and the clock is set to the specified
time on success, otherwise:

- -EINVAL if the time specification referred to by _ts_ is invalid
   (e.g. `ts->tv_nsec` not in the [0..1e9] range).

- -EBADF if _clockfd_ is neither a built-in clock identifier or a valid
   file descriptor.

- -EPERM if the caller does not have the permission to set some
   built-in clock via
   [clock_settime(3)](http://man7.org/linux/man-pages/man3/clock_settime.3.html). See
   note.

- -ENOTTY if _clockfd_ does not refer to an EVL clock device.

- -EFAULT if _tp_ points to invalid memory.

{{% notice note %}}
Setting `EVL_CLOCK_MONOTONIC` or `EVL_CLOCK_REALTIME` may imply a
transition to the [in-band execution stage]({{< relref
"dovetail/altsched.md#inband-switch" >}}) as
[clock_settime()](http://man7.org/linux/man-pages/man3/clock_settime.3.html)
is called internally for carrying out the request.
{{% /notice %}}

---

{{< proto evl_get_clock_resolution >}}
int evl_get_clock_resolution(int clockfd, struct timespec *tp)
{{< /proto >}}

This call is the EVL equivalent of the POSIX
[clock_getres(3)](http://man7.org/linux/man-pages/man3/clock_getres.3.html)
service, for reading the resolution of a specified EVL clock.  It
returns this value as counts of seconds and microseconds. The
resolution of clocks depends on the implementation and cannot be
configured.

{{% argument clockfd %}}
The clock file descriptor, which can be:

- any valid file descriptor received as a result of opening a clock
  device in _/dev/evl/clock_.

- the identifier of a [built-in EVL clock]({{< relref
  "#builtin-clocks" >}}), such as `EVL_CLOCK_MONOTONIC` or
  `EVL_CLOCK_REALTIME`.
{{% /argument %}}

{{% argument tp %}}
A pointer to a _timespec_ structure which should receive the resolution.
{{% /argument %}}

`evl_get_clock_resolution()` writes the resolution to _tp_ then
returns zero on success, otherwise:

- -EBADF if _clockfd_ is neither a built-in clock identifier or a valid
   file descriptor.

- -ENOTTY if _clockfd_ does not refer to an EVL clock device.

- -EFAULT if _tp_ points to invalid memory.

{{% notice note %}}
Setting `EVL_CLOCK_MONOTONIC` or `EVL_CLOCK_REALTIME` may imply a
transition to the [in-band execution stage]({{< relref
"dovetail/altsched.md#inband-switch" >}}) as
[clock_getres(3)](http://man7.org/linux/man-pages/man3/clock_getres.3.html)
is called internally for carrying out the request.
{{% /notice %}}

---

{{< proto evl_adjust_clock >}}
int evl_adjust_clock(int clockfd, struct timex *tx)
{{< /proto >}}

This call is the EVL equivalent of the Linux-specific
[adjtimex(2)](http://man7.org/linux/man-pages/man2/adjtimex.2.html)
service, for tuning the clock adjustment algorithm of a specified EVL
clock.

{{% argument clockfd %}}
The clock file descriptor, which must be a valid file descriptor
received as a result of opening a clock device in _/dev/evl/clock_.
{{% /argument %}}

{{% argument tp %}}
A pointer to a _timex_ structure which should define the tuning
parameters.
{{% /argument %}}

`evl_adjust_clock()` sets the tuning parameters from _tx_ then returns
zero on success, otherwise:

- -EBADF if _clockfd_ is neither a built-in clock identifier or a valid
   file descriptor.

- -EPERM if the caller does not have the permission to set the tuning
   parameters.

- -ENOTTY if _clockfd_ does not refer to an EVL clock device.

- -EINVAL if some information in _tx_ is invalid.

- -EFAULT if _tx_ points to invalid memory.

{{% notice note %}}
The EVL driver interfacing with the clock device must support
adjustment for this request to succeed.
{{% /notice %}}

---

{{< proto evl_sleep >}}
int evl_sleep(int clockfd, const struct timespec *timeout)
{{< /proto >}}

This call is the EVL equivalent of the POSIX
[clock_nanosleep(3)](http://man7.org/linux/man-pages/man3/clock_nanosleep.3.html)
service, for blocking the caller until an arbitrary date expires on a
specified EVL clock. Unlike its POSIX counterpart, `evl_sleep()` only
accepts absolute timeout specifications though.

{{% argument clockfd %}}
The clock file descriptor, which can be:

- any valid file descriptor received as a result of opening a clock
  device in _/dev/evl/clock_.

- the identifier of a [built-in EVL clock]({{< relref
  "#builtin-clocks" >}}), such as `EVL_CLOCK_MONOTONIC` or
  `EVL_CLOCK_REALTIME`.
{{% /argument %}}

{{% argument timeout %}}
A pointer to a _timespec_ structure which specifies the wake up date.
{{% /argument %}}

`evl_sleep()` blocks the caller until the wake up date expires then
returns zero on success, otherwise:

- -EBADF if _clockfd_ is neither a built-in clock identifier or a valid
   file descriptor.

- -ENOTTY if _clockfd_ does not refer to an EVL clock device.

- -EFAULT if _timeout_ points to invalid memory.

- -EINTR if the call was interrupted by an in-band signal, or forcibly
   unblocked by the EVL core.

---

{{< proto evl_udelay >}}
int evl_udelay(unsigned int usecs)
{{< /proto >}}

This call puts the caller to sleep until a count of microseconds has
elapsed. `evl_udelay()` invokes [evl_sleep()]({{< relref "#evl_sleep"
>}}) for sleeping until the delay expires on the EVL_CLOCK_MONOTONIC
clock.

{{% argument usecs %}}
The count of microseconds to sleep for.
{{% /argument %}}

`evl_udelay()` blocks the caller until the delay expires then returns
zero on success, otherwise:

- -EINTR if the call was interrupted by an in-band signal, or forcibly
   unblocked by the EVL core.

### Pre-defined clocks {#builtin-clocks}

EVL defines two built-in clocks, you can pass any of the following
identifiers to EVL calls which ask for a clock file descriptor
(usually noted as _clockfd_):

- `EVL_CLOCK_MONOTONIC` is identical to the `CLOCK_MONOTONIC` POSIX
  clock, which is a monotonically increasing clock that cannot be set
  and represents time since some unspecified starting point (aka _the
  epoch_). This identifier has the same meaning than a file descriptor
  opened on _/dev/evl/clock/monotonic_.

- `EVL_CLOCK_REALTIME` is identical to the `CLOCK_REALTIME` POSIX
  clock, which is a non-monotonic wall clock which can be manually set
  to an arbitrary value with proper privileges, and can also be
  subject to dynamic adjustements by the NTP system. This identifier
  has the same meaning than a file descriptor opened on
  _/dev/evl/clock/realtime_.

If you are to measure the elapsed time between two events, you
definitely want to use `EVL_CLOCK_MONOTONIC`.
