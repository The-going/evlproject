---
menuTitle: "Timer(fd)"
weight: 30
---

### Running oneshot or periodic timers

At some point, you will most likely need to synchronize some thread
with ultra-low latency clock events, either once or according to a
recurring period. The EVL core gives you timers for this purpose,
which can be obtained on any existing [EVL clock]({{< relref
"core/user-api/clocks/_index.md" >}}). EVL timers support synchronous
delivery, there is no asynchronous notification mechanism of clock
events. This means that a client thread must explicitly wait for the
next expiry by either [reading]({{< relref
"core/user-api/io/_index.md#oob_read" >}}) or [polling]({{< relref
"#timer-poll-events" >}}) for such event.  Would you choose to
[poll]({{< relref "#timer-poll-events" >}}) for clock events on a
timer, you would still have to collect these events once they have
occurred by [reading]({{< relref "core/user-api/io/_index.md#oob_read"
>}}) the timer file descriptor.

Each EVL timer is referred to by a file descriptor. The way you would
use those timers mimics the usage of the [timerfd
API](http://man7.org/linux/man-pages/man2/timerfd_create.2.html) for
the most part, which consists of:

1. creating a timer 2. setting up a timeout date, either oneshot or
recurring 3. waiting for the (next) timeout to elapse by calling
[oob_read()]({{< relref "core/user-api/io/_index.md#oob_read" >}}) for
the timer file descriptor. The value collected by reading a timer is
the number of elapsed ticks since the last readout. Therefore, any
value greater than 1 would denote an overrun condition.

> A simple periodic loop using an EVL timer

```
	#include <evl/thread.h>
	#include <evl/timer.h>
	#include <evl/clock.h>
	#include <evl/proxy.h>

	void timespec_add_ns(struct timespec *__restrict r,
	     		     const struct timespec *__restrict t,
			     long ns)
	{
		long s, rem;

		s = ns / 1000000000;
		rem = ns - s * 1000000000;
		r->tv_sec = t->tv_sec + s;
		r->tv_nsec = t->tv_nsec + rem;
		if (r->tv_nsec >= 1000000000) {
		     r->tv_sec++;
		     r->tv_nsec -= 1000000000;
		}
	}

	int main(int argc, char *argv[])
	{
		struct itimerspec value, ovalue;
		int tfd, tmfd, ret, n = 0;
		struct timespec now;
		__u64 ticks;

		/* Attach to the core. */
		tfd = evl_attach_self("periodic-timer:%d", getpid());
		check_this_fd(tfd);

		/* Create a timer on the built-in monotonic clock. */
		tmfd = evl_new_timer(EVL_CLOCK_MONOTONIC);
		check_this_fd(tmfd);

		/* Set up a 1 Hz periodic timer. */
		ret = evl_read_clock(EVL_CLOCK_MONOTONIC, &now);
		check_this_status(ret);
		/* EVL always uses absolute timeouts, add 1s to the current date */
		timespec_add_ns(&value.it_value, &now, 1000000000ULL);
		value.it_interval.tv_sec = 1;
		value.it_interval.tv_nsec = 0;
		ret = evl_set_timer(tmfd, &value, &ovalue);
		check_this_status(ret);

		for (;;) {
		    /* Wait for the next tick to be notified. */
		    ret = oob_read(tmfd, &ticks, sizeof(ticks));
		    check_this_status(ret);
		    if (ticks > 1) {
		       	      fprintf(stder, "timer overrun! %lld ticks late\n",
			      	      ticks - 1);
			      break;
		    }
		    evl_printf("TICKED, loops=%d\n", n++);
		}

		/* Disable the timer (not required if closing). */
		value.it_interval.tv_sec = 0;
		value.it_interval.tv_nsec = 0;
		ret = evl_set_timer(tmfd, &value, NULL);
		check_this_status(ret);

		return 0;
}
```

### Timer services

{{< proto evl_new_timer >}}
int evl_new_timer(int clockfd)
{{< /proto >}}

This call creates an EVL timer on the clock referred to by _clockfd_,
returning a file descriptor representing the new object upon
success. There is no arbitrary limit on the number of timers an
application can create, which is only limited to the available system
resources.

{{% argument clockfd %}}
The file descriptor of the reference clock the new timer should be
synchronized on. _clockfd_ refer to any valid [clock element]({{<
relref "core/user-api/clock/_index.md" >}}) known from the EVL core,
including one of the [pre-defined clocks]({{< relref
"core/user-api/clock/_index.md#builtin-clocks" >}}).
{{% /argument %}}

`evl_new_timer()` returns the file descriptor of the newly created
timer on success. Otherwise, a negated error code is returned:

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

- -ENXIO	The EVL library is not initialized for the current process.
  		Such initialization happens implicitly when
  		[evl_attach_self()]({{% relref
  		"core/user-api/thread/_index.md#evl_attach_self" %}})
  		is called by any thread of your process, or by
  		explicitly calling [evl_init()]({{< relref
  		"core/user-api/init/_index.md#evl_init" >}}). You have
  		to bootstrap the library services in a way or another
  		before creating an EVL timer.

---

{{< proto evl_set_timer >}}
int evl_set_timer(int efd, struct itimerspec *value, struct itimerspec *ovalue)
{{< /proto >}}

You should use this call to arm or disarm an EVL timer. It sets the
next expiration date and reload value for the timer referred to by
_efd_. If _ovalue_ is not NULL, the current expiration date and reload
value are stored at this location prior to updating them, as with
[evl_get_timer]({{< relref "#evl_get_timer" >}}).

{{% argument efd %}}
A file descriptor referring to the EVL timer to set up.
{{% /argument %}}

{{% argument value %}}
A pointer to the interval timer specification structure which contains
the new **absolute** expiration date and reload value.  If both
`value->it_value.tv_nsec` and `value->it_value.tv_sec` are zero, the
timer is disarmed and `value->it_interval` is ignored. Otherwise, the
timer is program to timeout at the specified date mentioned in
`value->it_value`. If at least on field of `value->it_interval` is
non-zero, the EVL will reprogram the timer after each timeout
according to this period automatically.
{{% /argument %}}

{{% argument ovalue %}}
If non-NULL, the previous expiration date and reload value of the timer
are copied to this memory location.
{{% /argument %}}

Zero is returned on success.  Otherwise, a negated error code is
returned:

- -EBADF	 _efd_ is not a valid file descriptor referring to an EVL
  		 timer.

- -EINVAL 	 some field in *_value_ is not valid, likely
  	     	 `it_value.tv_nsec` and/or `it_interval.tv_nsec`
		 are out of the valid range, which is between 0
		 and 1e9 - 1.

- -EFAULT	 either _value_ or _ovalue_ (if non-NULL) point to invalid
  		 memory.

---

{{< proto evl_get_timer >}}
int evl_get_timer(int efd, struct itimerspec *value)
{{< /proto >}}

Retrieve the current timeout and reload values of an EVL timer.

{{% argument efd %}}
A file descriptor referring to the EVL timer to query.
{{% /argument %}}

{{% argument value %}}
The current expiration date and reload value of the timer are copied
to this memory location. The expiration date is **always** given in
**absolute** form based on the reference clock for the timer.
{{% /argument %}}

Zero is returned on success.  Otherwise, a negated error code is
returned:

- -EBADF	 _efd_ is not a valid file descriptor referring to an EVL
  		 timer.

- -EFAULT	 if _value_ points to invalid memory.

---

### Events pollable from a timer file descriptor {#timer-poll-events}

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}})
interface can monitor the following events occurring on a timer
file descriptor:

- _POLLIN_ and _POLLRDNORM_ are set whenever a timeout event has fired
  for the timer, which is yet to be collected by the parent
  process. Timer events should be awaited for and collected by a call
  to [oob_read()]({{< relref "core/user-api/io/_index.md#oob_read"
  >}}).

---

{{<lastmodified>}}
