---
menuTitle: "Event"
weight: 15
---

### Synchronizing on arbitrary events {#synchronizing-on-event}

An EVL event object has the same semantics than a POSIX [condition
variable](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_cond_wait.html). It
can be used for synchronizing EVL threads on the state of some shared
information in a raceless way. Proper serialization is enforced by
pairing an event with a [mutex]({{% relref
"core/user-api/mutex/_index.md" %}}). The typical usage pattern is as
follows:

```
#include <evl/mutex.h>
#include <evl/event.h>

struct evl_mutex lock;
struct evl_event event;
bool condition = false;

void waiter(void)
{
	evl_lock_mutex(&lock);
	/*
	 * Check whether the wait condition is satisfied, go
	 * sleeping if not. EVL drops the lock and puts the thread
	 * to sleep atomically, then re-acquires it automatically
	 * once the event is signaled before returning to the
	 * caller.
	 */
	while (!condition)
	      evl_wait_event(&event, &lock);

	evl_unlock_mutex(&lock);
}
	
void poster(void)
{
	/*
	 * Post the condition and signal the event to the waiter,
	 * all atomically by holding the lock. CAUTION: actual wake
	 * up happens when dropping the lock protecting the event.
	 */
	evl_lock_mutex(&lock);
	condition = true;
	evl_signal_event(&event);
	evl_unlock_mutex(&lock);
}
	
```

Of course, the way you would express the condition to be satisfied is
fully yours. It could be a simple boolean condition like illustrated,
or the combination of multiple tests and/or computations, whatever
works.

{{% notice info %}}
Unlike _glibc_ with condition variables, EVL optimizes the wake up
sequence on event receipt by readying the waiter only upon release of
the lock which protects accesses to the condition by the thread
posting the awaited event. This spares us a useless context switch to
the waiter in case it has a higher priority than the poster.
{{% /notice %}}

### Event services {#event-services}

---

{{< proto evl_new_event >}}
int evl_new_event(struct evl_event *evt, int clockfd, const char *fmt, ...)
{{< /proto >}}

To create an event, you need to call `evl_new_event()` which returns a
file descriptor representing the new object upon success.

{{% argument evt %}}
An in-memory event descriptor is constructed by `evl_new_event()`,
which contains ancillary information other calls will need. _evt_
is a pointer to such descriptor of type `struct evl_event`.
{{% /argument %}}

{{% argument clockfd %}}
Some event-related calls are timed like `evl_timedwait_event()` which
receives a timeout value. You can specify the [EVL clock]({{% relref
"core/user-api/clock/_index.md" %}}) this timeout refers to by passing
its file descriptor as _clockfd_. [Built-in EVL clocks]({{% relref
"core/user-api/clock/_index.md#builtin-clocks" %}}) are accepted here.
{{% /argument %}}

{{% argument fmt %}}
A printf-like format string to generate the event name. A common way
of generating unique event names is to add the calling process's
_pid_ somewhere into the format string as illustrated in the
example. The generated name is used to form a last part of a pathname,
referring to the new event element's device in the file system. So
this name must contain only valid characters in this context,
excluding slashes.
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

`evl_new_event()` returns the file descriptor of the newly created
event on success. Otherwise, a negated error code is returned:

- -EEXIST	The generated name is conflicting with an existing mutex,
		event, semaphore or flag group name.

- -EINVAL	Either _clockfd_ does not refer to a valid EVL clock, or the
  		generated event name is badly formed, likely containing
		invalid character(s), such as a slash. Keep in mind that
		it should be usable as a basename of a device element's file path.

- -ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

- -ENXIO        The EVL library is not initialized for the current process.
  		Such initialization happens implicitly when `evl_attach_self()`
		is called by any thread of your process, or by explicitly
		calling `evl_init()`. You have to bootstrap the library
		services in a way or another before creating an EVL event.

{{% notice info %}}
An event object cannot be used alone, it has to be paired with an [EVL
mutex]({{% relref "core/mutex/_index.md" %}}) for proper operation.
{{% /notice %}}

```
#include <evl/event.h>

static struct evl_event event;

void create_new_event(void)
{
	int evfd;

	evfd = evl_new_event(&event, EVL_CLOCK_MONOTONIC, "name_of_event");
	/* skipping checks */
	
	return evfd;
}
```

An event is based on the [EVL monitor]({{% relref
"core/_index.md#evl-core-elements" %}}) element. For this reason, the
device representing the new element appears in the _/dev/evl/monitor/_
hierarchy.

---

{{< proto EVL_EVENT_INITIALIZER >}}
EVL_EVENT_INITIALIZER(name, clockfd)
{{< /proto >}}

The static initializer you can use with events.

{{% argument name %}}
A name which is used to form a last part of a pathname, referring to
the new event element's device in the file system. So this name must
contain only valid characters in this context, excluding slashes.
{{% /argument %}}

{{% argument clockfd %}}
The [EVL clock]({{% relref "core/user-api/clock/_index.md" %}})
associated with the event for timed operations. The descriptors of
[built-in EVL clocks]({{% relref
"core/user-api/clock/_index.md#builtin-clocks" %}}) are constant
values which are accepted here.
{{% /argument %}}

---

{{< proto evl_open_event >}}
int evl_open_event(struct evl_event *evt, const char *fmt, ...)
{{< /proto >}}

Knowing its name in the EVL device hierarchy, you may want to open
an existing event, possibly from a different process, obtaining a new
file descriptor on such object. You can do so by calling
`evl_open_event()`.

{{% argument evt %}}
An in-memory event descriptor is constructed by `evl_open_event()`,
which contains ancillary information other calls will need. _evt_ is a
pointer to such descriptor of type `struct evl_event`. The information
is retrieved from the existing event which was opened.
{{% /argument %}}

{{% argument fmt %}}
A printf-like format string to generate the name of the event to
open. This name must exist in the EVL device hierachy at
_/dev/evl/monitor/_.
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

`evl_open_event()` returns the file descriptor referring to the opened
event on success, Otherwise, a negated error code is returned:

- -EINVAL	The name refers to an existing object, but not to an event.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

---

{{< proto evl_wait_event >}}
int evl_wait_event(struct evl_event *evt, struct evl_mutex *mutex)
{{< /proto >}}

You should use this call to [wait for the condition]({{% relref
"#synchronizing-on-event" %}}) associated with an EVL event to become
true. Access to such condition must be protected by an [EVL mutex]({{%
relref "core/mutex/_index.md" %}}). The caller is put to sleep by the
core until this happens, dropping the lock protecting the condition in
the same move, all atomically so that no wake up event can be
missed. Waiters are queued by order of [scheduling priority]({{<
relref "core/user-api/scheduling/_index.md" >}}).

Keep in mind that events may be subject to spurious wake ups, meaning
that the condition might turn to false again before the waiter can
resume execution for consuming the event. Therefore, abiding by the
documented [usage pattern]({{% relref "#synchronizing-on-event" %}})
for waiting for an event is very important.

{{% argument evt %}}
The in-memory event descriptor constructed by
either `evl_new_event()` or `evl_open_event()`, or statically built
with [EVL_EVENT_INITIALIZER()]({{% relref "#EVL_EVENT_INITIALIZER"
%}}). In the latter case, an implicit call to `evl_new_event()` for
_evt_ is issued before a wait is attempted, which may trigger a transition
to the in-band execution mode for the caller.
{{% /argument %}}

{{% argument mutex %}}
The in-memory mutex descriptor returned by either `evl_new_mutex()` or
`evl_open_mutex()`. This lock should be used to serialize access to the
shared information representing the condition.
{{% /argument %}}

`evl_wait_event()` returns zero if the event was signaled by a call to
`evl_signal_event()` or `evl_broadcast_event()`. Otherwise, a negated
error code may be returned if:

-EINVAL	      _evt_ does not represent a valid in-memory event
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

-EBADFD	      A thread is currently pending on _evt_ protected by a
	      different mutex. When multiple threads issue concurrent
	      wait requests on the same event, they must use the same
	      mutex to serialize.

If _evt_ was statically initialized with EVL_EVENT_INITIALIZER(), then
any error returned by [evl_new_event()]({{% relref "#evl_new_event"
%}}) may be passed back to the caller in case the implicit
initialization call fails.

---

{{< proto evl_timedwait_event >}}
int evl_timedwait_event(struct evl_event *evt, struct evl_mutex *mutex, const struct timespec *timeout)
{{< /proto >}}

This call is a variant of [evl_wait_event()]({{% relref
"#evl_wait_event" %}}) which allows specifying a timeout on the wait
operation, so that the caller is unblocked after a specified delay
sleeping without the event being signaled.

{{% argument evt %}}
The in-memory event descriptor constructed by
either `evl_new_event()` or `evl_open_event()`, or statically built
with [EVL_EVENT_INITIALIZER()]({{% relref "#EVL_EVENT_INITIALIZER"
%}}). In the latter case, an implicit call to `evl_new_event()` for
_evt_ is issued before a wait is attempted, which may trigger a transition
to the in-band execution mode for the caller.
{{% /argument %}}

{{% argument mutex %}}
The in-memory mutex descriptor returned by either `evl_new_mutex()` or
`evl_open_mutex()`.
{{% /argument %}}

{{% argument timeout %}}
A time limit to wait for the event to be signaled before the call
returns on error. The clock mentioned in the call to
[evl_new_event()]({{% relref "#evl_new_event" %}}) will be used for
tracking the elapsed time.
{{% /argument %}}

The possible return values include any status from
[evl_wait_event()]({{% relref "#evl_wait_event" %}}), plus:

-ETIMEDOUT	   The timeout fired, after the amount of time specified by
		   _timeout_.

---

{{< proto evl_signal_event >}}
int evl_signal_event(struct evl_event *evt)
{{< /proto >}}

This call notifies the thread heading the wait queue of an event at
the time of the call (i.e. having the highest priority among waiters),
indicating that the condition it has waited for may have become true.

You must hold the _mutex_ lock which protects accesses to the
condition around the call to `evl_signal_event()`, because the
notified thread will be actually [readied when such lock is
dropped]({{< relref "#synchronizing-on-event" >}}), not at the time of
the notification.

{{% argument evt %}}
The in-memory event descriptor constructed by either `evl_new_event()`
or `evl_open_event()`, or statically built with
[EVL_EVENT_INITIALIZER()]({{% relref "#EVL_EVENT_INITIALIZER" %}}). In
the latter case, an implicit call to `evl_new_event()` for _evt_ is
issued before the signal is sent, which may trigger a transition to
the in-band execution mode for the caller.
{{% /argument %}}

`evl_signal_event()` returns zero upon success, with the signal set
pending for the event. Otherwise, a negated error code is returned:

-EINVAL	      _evt_ does not represent a valid in-memory event
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

If _evt_ was statically initialized with EVL_EVENT_INITIALIZER() but
not passed to any event-related call yet, then any error status
returned by [evl_new_event()]({{% relref "#evl_new_event" %}}) may be
passed back to the caller in case the implicit initialization call
fails.

If no thread was waiting on _evt_ at the time of the call, no action
is performed and `evl_signal_event()` returns zero.

---

{{< proto evl_signal_thread >}}
int evl_signal_thread(struct evl_event *evt, int thrfd)
{{< /proto >}}

This service is a variant of [evl_signal_event()]({{% relref
"#evl_signal_event" %}}), which directs the wake up signal to a
specific waiter instead of waking up the thread heading the wait
queue.

{{% argument evt %}}
The in-memory event descriptor constructed by either `evl_new_event()`
or `evl_open_event()`, or statically built with
[EVL_EVENT_INITIALIZER()]({{% relref "#EVL_EVENT_INITIALIZER" %}}). In
the latter case, an implicit call to `evl_new_event()` for _evt_ is
issued before the signal is sent, which may trigger a transition to
the in-band execution mode for the caller.
{{% /argument %}}

{{% argument thrfd %}}
The file descriptor of the target thread, which must be attached to
the EVL core by a call to [evl_attach_self()]({{% relref
"core/user-api/thread/_index.md" %}}).
{{% /argument %}}

The possible return values include any status from
[evl_signal_event()]({{% relref "#evl_signal_event" %}}), plus:

-EINVAL		   _thrfd_ is not a valid file descriptor referring
		   to an EVL thread.

If the target thread was not waiting on _evt_ at the time of the call,
no action is performed and `evl_signal_thread()` returns zero.

---

{{< proto evl_broadcast_event >}}
int evl_broadcast(struct evl_event *evt)
{{< /proto >}}

This service is yet another variant of [evl_signal_event()]({{% relref
"#evl_signal_event" %}}), which wakes up _all_ waiters currently
sleeping on the event wait queue at the time of the call.

{{% argument evt %}}
The in-memory event descriptor constructed by either `evl_new_event()`
or `evl_open_event()`, or statically built with
[EVL_EVENT_INITIALIZER()]({{% relref "#EVL_EVENT_INITIALIZER" %}}). In
the latter case, an implicit call to `evl_new_event()` for _evt_ is
issued before the signal is sent, which may trigger a transition to
the in-band execution mode for the caller.
{{% /argument %}}

The possible return values include any status from
[evl_signal_event()]({{% relref "#evl_signal_event" %}}).

If no thread was waiting on _evt_ at the time of the call, no action
is performed and `evl_signal_broadcast()` returns zero.

---

{{< proto evl_close_event >}}
int evl_close_event(struct evl_event *evt)
{{< /proto >}}

You can use `evl_close_event()` to dispose of an EVL event, releasing
the associated file descriptor, at which point _evt_ will not be valid
for any subsequent operation from the current process. However, this
event object is kept alive in the EVL core until all file descriptors
opened on it by call(s) to [evl_open_event()]({{< relref
"#evl_open_event" >}}) have been released, whether from the current
process or any other process.

{{% argument evt %}}
The in-memory descriptor of the event to dismantle.
{{% /argument %}}

`evl_close_event()` returns zero upon success. Otherwise, a negated
error code is returned:

-EINVAL	      _evt_ does not represent a valid in-memory event
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

Closing a [statically initialized]({{< relref "#EVL_EVENT_INITIALIZER"
>}}) event descriptor which has never been used in wait or signal
operations always returns zero.
