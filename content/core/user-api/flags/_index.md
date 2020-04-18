---
menuTitle: "Flags"
weight: 17
---

### Synchronizing on a group of flags {#synchronizing-on-flags}

An event flag group is an efficient and lightweight mechanism for
synchronizing multiple threads, based on a 32bit value, in which each
individual bit represents the state of a particular event defined by
the application. Therefore, each group defines 32 individual events,
from bit #0 to bit #31. Threads can wait for bits to be posted by
other threads. Although EVL's event flag group API is somewhat
reminiscent of the [eventfd
interface](http://man7.org/linux/man-pages/man2/eventfd.2.html), it is
much simpler:

- on the send side, posting the _bits_ event mask to an event _group_
  is equivalent to performing atomically a bitwise OR operation such
  as _group |= bits_.
 
- on the receive side, waiting for events means blocking until _group_
  becomes non-zero, at which point this value is returned to the
  thread heading the wait queue and the group is reset to zero in the
  same move. In other words, all bits set are immediately consumed by
  the first waiter and cleared in the group atomically. Threads wait
  on a group by priority order, which is determined by the [scheduling
  policy]({{< relref "core/user-api/scheduling/_index.md" >}}) they
  undergo.

![Alt text](/images/flags.png "Event flag group")

{{% notice tip %}}
Unlike with the
[eventfd](http://man7.org/linux/man-pages/man2/eventfd.2.html), there
is no semaphore semantics associated to an event flag group, you may
want to consider the EVL [semaphore]({{< relref
"core/user-api/semaphore/_index.md" >}}) feature instead if this is
what you are looking for.
{{% /notice %}}

### Event flag group services

---

{{< proto evl_create_flags >}}
int evl_create_flags(struct evl_flags *flg, int clockfd, init initval, int flags, const char *fmt, ...)
{{< /proto >}}

This call creates a group of event flags, returning a file descriptor
representing the new object upon success.  This is the generic call
form; for creating an event group with common pre-defined settings,
see [evl_new_flags()}({{% relref "#evl_new_flags" %}}).

{{% argument flg %}}
An in-memory flag group descriptor is constructed by
[evl_create_flags()]({{< relref "#evl_create_flags" >}}),
which contains ancillary information other calls will need. _flg_
is a pointer to such descriptor of type `struct evl_flags`.
{{% /argument %}}

{{% argument clockfd %}}
Some flag group-related calls are timed like
[evl_timewait_flags()]({{< relref "#evl_timedwait_flags" >}})
which receives a timeout value. You can specify the [EVL clock]({{%
relref "core/user-api/clock/_index.md" %}}) this timeout refers to by
passing its file descriptor as _clockfd_. [Built-in EVL clocks]({{%
relref "core/user-api/clock/_index.md#builtin-clocks" %}}) are
accepted here.
{{% /argument %}}

{{% argument initval %}}
The initial value of the flag group. You can use this parameter to
pre-set some bits in the received event mask at creation time.
{{% /argument %}}

{{% argument flags %}}
A set of creation flags for the new element, defining its
[visibility]({{< relref "core/user-api/_index.md#element-visibility"
>}}):

  - `EVL_CLONE_PUBLIC` denotes a public element which is represented
    by a device file in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file
    hierarchy, which makes it visible to other processes for sharing.
  
  - `EVL_CLONE_PRIVATE` denotes an element which is private to the
    calling process. No device file appears for it in the
    [/dev/evl]({{< relref "core/user-api/_index.md#evl-fs-hierarchy" >}})
    file hierarchy.
 {{% /argument %}}

{{% argument fmt %}}
A [printf](http://man7.org/linux/man-pages/man3/printf.3.html)-like
format string to generate the flag group name. See this description of the
[naming convention]
({{< relref "core/user-api/_index.md#element-naming-convention" >}}).
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_create_flags()]({{< relref "#evl_create_flags" >}}) returns the file descriptor of the newly created
flag group on success. Otherwise, a negated error code is returned:

- -EEXIST	The generated name is conflicting with an existing mutex,
		event, semaphore or flag group name.

- -EINVAL	Either _clockfd_ does not refer to a valid EVL clock, or the
  		generated group name is badly formed, likely containing
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
  		Such initialization happens implicitly when [evl_attach_self()]({{% relref
  		"core/user-api/thread/_index.md#evl_attach_self" %}})
		is called by any thread of your process, or by explicitly
		calling [evl_init()]({{<
  		relref "core/user-api/init/_index.md#evl_init"
  		>}}). You have to bootstrap the library
		services in a way or another before creating an EVL flag group.

```
#include <evl/flags.h>

static struct evl_flags flags;

void create_new_flags(void)
{
	int fd;

	fd = evl_create_flags(&flags, EVL_CLOCK_MONOTONIC, 0, "name_of_group");
	/* skipping checks */
	
	return fd;
}
```

---

{{< proto evl_new_flags >}}
int evl_new_flags(struct evl_flags *flg, const char *fmt, ...)
{{< /proto >}}

This call is a shorthand for creating a zero-initialized group of
event flags, timed on the built-in [EVL monotonic clock]({{% relref
"core/user-api/clock/_index.md#builtin-clocks" %}}). It is identical
to calling:

```
	evl_create_flags(flg, EVL_CLOCK_MONOTONIC, 0, EVL_CLONE_PRIVATE, fmt, ...);
```

{{% notice info %}}
Note that if the [generated name] ({{< relref
"core/user-api/_index.md#element-naming-convention" >}}) starts with a
slash ('/') character, `EVL_CLONE_PRIVATE` would be automatically turned
into `EVL_CLONE_PUBLIC` internally.
{{% /notice %}}

---

{{< proto EVL_FLAGS_INITIALIZER >}}
EVL_FLAGS_INITIALIZER((const char *) name, (int) clockfd, (int) initval, (int) flags)
{{< /proto >}}

The static initializer you can use with flag groups. All arguments to
this macro refer to their counterpart in the call to
[evl_create_flags()]({{< relref "#evl_create_flags" >}}).

---

{{< proto evl_open_flags >}}
int evl_open_flags(struct evl_flags *flg, const char *fmt, ...)
{{< /proto >}}

You can open an existing flag group, possibly from a different
process, by calling [evl_open_flags()]({{< relref "#evl_open_flags" >}}).

{{% argument flg %}}
An in-memory flag group descriptor is constructed by [evl_open_flags()]({{< relref "#evl_open_flags" >}}),
which contains ancillary information other calls will need. _flg_ is a
pointer to such descriptor of type `struct evl_flags`. The information
is retrieved from the existing flag group which was opened.
{{% /argument %}}

{{% argument fmt %}}
A printf-like format string to generate the name of the flag group to
open. This name must exist in the EVL device file hierarchy at
[/dev/evl/monitor]({{< relref
"core/user-api/_index.md#evl-fs-hierarchy" >}}).
See this description of the [naming convention]({{<
relref "core/user-api/_index.md#element-naming-convention" >}}).
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_open_flags()]({{< relref "#evl_open_flags" >}}) returns the file descriptor referring to the opened
group on success, Otherwise, a negated error code is returned:

- -EINVAL	The name refers to an existing object, but not to a group.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

---

{{< proto evl_wait_flags >}}
int evl_wait_flags(struct evl_flags *flg, int *r_bits)
{{< /proto >}}

This service [waits for events]({{% relref "#synchronizing-on-flags"
%}}) to be posted to the flag group.  The caller is put to sleep by
the core until this happens. Waiters are queued by order of
[scheduling priority]({{< relref "core/user-api/scheduling/_index.md"
>}}).

When the group receives some event(s), its value is passed back to the
waiter leading the wait queue, then reset to zero before the latter
returns.

If the flag group value is non-zero on entry to this service, the
caller returns immediately without blocking with the current set of
posted flags.

{{% argument flg %}}
The in-memory flag group descriptor constructed by
either [evl_create_flags()]({{< relref "#evl_create_flags" >}}) or
[evl_open_flags()]({{< relref "#evl_open_flags" >}}), or statically built
with [EVL_FLAGS_INITIALIZER]({{% relref "#EVL_FLAGS_INITIALIZER"
%}}). In the latter case, an implicit call to
[evl_create_flags()]({{< relref "#evl_create_flags" >}}) is
issued for _flg_ before a wait is attempted, which may trigger a transition
to the in-band execution mode for the caller.
{{% /argument %}}

{{% argument r_bits %}}
The address of an integer which contains the set of flags received on
successful return from the call.
{{% /argument %}}

[evl_wait_flags()]({{< relref "#evl_wait_flags" >}}) returns zero on
success. Otherwise, a negated error code may be returned if:

-EINVAL	      _flg_ does not represent a valid in-memory flag group
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

If _flg_ was statically initialized with [EVL_FLAGS_INITIALIZER]({{%
relref "#EVL_FLAGS_INITIALIZER" %}}), then any error returned by
[evl_create_flags()]({{% relref "#evl_create_flags" %}}) may be passed
back to the caller in case the implicit initialization call fails.

---

{{< proto evl_timedwait_flags >}}
int evl_timedwait_flags(struct evl_flags *flg, const struct timespec *timeout, int *r_bits)
{{< /proto >}}

This call is a variant of [evl_wait_flags()]({{% relref
"#evl_wait_flags" %}}) which allows specifying a timeout on the wait
operation, so that the caller is unblocked after a specified delay
sleeping without any flag being posted.

{{% argument flg %}}
The in-memory flag group descriptor constructed by
either [evl_create_flags()]({{< relref "#evl_create_flags" >}})
or [evl_open_flags()]({{< relref "#evl_open_flags" >}}), or statically built
with [EVL_FLAGS_INITIALIZER]({{% relref "#EVL_FLAGS_INITIALIZER"
%}}). In the latter case, an implicit call to
[evl_create_flags()]({{< relref "#evl_create_flags" >}}) for
_flg_ is issued before a wait is attempted, which may trigger a transition
to the in-band execution mode for the caller.
{{% /argument %}}

{{% argument timeout %}}
A time limit to wait for the group to be posted before the call
returns on error. The clock mentioned in the call to
[evl_create_flags()]({{% relref "#evl_create_flags" %}}) will be used for
tracking the elapsed time.
{{% /argument %}}

{{% argument r_bits %}}
The address of an integer which contains the set of flags received on
successful return from the call.
{{% /argument %}}

The possible return values include any status from
[evl_wait_flags()]({{% relref "#evl_wait_flags" %}}), plus:

-ETIMEDOUT	   The timeout fired, after the amount of time specified by
		   _timeout_.

---

{{< proto evl_post_flags >}}
int evl_post_flags(struct evl_flags *flg, int bits)
{{< /proto >}}

This call posts a (non-null) set of events to a flag group. The core
adds the posted bits to the group's current value using a bitwise OR
operation. Afterwards, it unblocks the thread heading the wait queue
of the flag group at the time of the call if any (i.e. having the
highest priority among waiters).

{{% argument flg %}}
The in-memory flag group descriptor constructed by either
[evl_create_flags()]({{< relref "#evl_create_flags" >}})
or [evl_open_flags()]({{< relref "#evl_open_flags" >}}), or statically built with
[EVL_FLAGS_INITIALIZER]({{% relref "#EVL_FLAGS_INITIALIZER" %}}). In
the latter case, an implicit call to
[evl_create_flags()]({{< relref "#evl_create_flags" >}}) for _flg_ is
issued before the event mask is posted, which may trigger a transition to
the in-band execution mode for the caller.
{{% /argument %}}

{{% argument bits %}}
The non-zero bitmask to add to the flag group value.
{{% /argument %}}

[evl_post_flags()]({{< relref "#evl_post_flags" >}}) returns zero upon
success. Otherwise, a negated error code is returned:

-EINVAL	      _flg_ does not represent a valid in-memory flag group
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

-EINVAL	      _bits_ is zero.

If _flg_ was statically initialized with
[EVL_FLAGS_INITIALIZER]({{% relref
"#EVL_FLAGS_INITIALIZER" %}}) but not passed to any flag
group-related call yet, then any error status returned by
[evl_create_flags()]({{% relref "#evl_create_flags" %}}) may be passed back
to the caller in case the implicit initialization call fails.

---

{{< proto evl_trywait_flags >}}
int evl_trywait_flags(struct evl_flags *flg, int *r_bits)
{{< /proto >}}

This call attempts to consume the event flags currently posted to a
group without blocking the caller if there are none. If some events
were pending at the time of the call, the group value is reset to zero
before returning to the caller.

{{% argument flg %}}
The in-memory flag group descriptor constructed by
either [evl_create_flags()]({{< relref "#evl_create_flags" >}})
or [evl_open_flags()]({{< relref "#evl_open_flags" >}}), or statically built
with [EVL_FLAGS_INITIALIZER]({{% relref "#EVL_FLAGS_INITIALIZER" %}}).
In the latter case, an implicit call to
[evl_create_flags()]({{< relref "#evl_create_flags" >}}) for
_flg_ is issued before a wait is attempted, which may trigger a transition
to the in-band execution mode for the caller.
{{% /argument %}}

{{% argument r_bits %}}
The address of an integer which contains the set of flags received on
successful return from the call.
{{% /argument %}}

[evl_trywait_flags()]({{< relref "#evl_trywait_flags" >}}) returns
zero on success and the non-zero set of pending events. Otherwise, a
negated error code may be returned if:

-EAGAIN       _flg_ had no event pending at the time of the call.

-EINVAL	      _flg_ does not represent a valid in-memory flag group
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

If _flg_ was statically initialized with
[EVL_FLAGS_INITIALIZER]({{% relref
"#EVL_FLAGS_INITIALIZER" %}}), then any error returned by
[evl_create_flags()]({{% relref "#evl_create_flags" %}}) may be passed
back to the caller in case the implicit initialization call fails.

---

{{< proto evl_peek_flags >}}
int evl_peek_flags(struct evl_flags *flg, int *r_bits)
{{< /proto >}}

This call is a variant of [evl_trywait_flags()]({{% relref
"#evl_trywait_flags" %}}) which does not consume the flags before
returning to the caller. In other words, the group value is not reset
to zero before returning a non-zero set of pending events, allowing
the group value to be read multiple times with no side-effect.

{{% argument flg %}}
The in-memory flag group descriptor constructed by either
[evl_create_flags()]({{< relref "#evl_create_flags" >}})
or [evl_open_flags()]({{< relref "#evl_open_flags" >}}), or statically built with
[EVL_FLAGS_INITIALIZER]({{% relref "#EVL_FLAGS_INITIALIZER" %}}). In
the latter case, the flag group becomes valid for a call to
[evl_peek_flags()]({{< relref "#evl_peek_flags" >}})
only after a post or [try]wait operation was issued
for it.
{{% /argument %}}

{{% argument r_bits %}}
The address of an integer which contains the group value on
successful return from the call.
{{% /argument %}}

[evl_peek_flags()]({{< relref "#evl_peek_flags" >}}) returns zero on
success along with the current group value. Otherwise, a negated error
code may be returned if:

-EINVAL	      _flg_ does not represent a valid in-memory flag group
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

---

{{< proto evl_close_flags >}}
int evl_close_flags(struct evl_flags *flg)
{{< /proto >}}

You can use [evl_close_flags()]({{< relref "#evl_close_flags" >}}) to
dispose of an EVL flag group, releasing the associated file
descriptor, at which point _flg_ will not be valid for any subsequent
operation from the current process. However, this flag group is kept
alive in the EVL core until all file descriptors opened on it by
call(s) to [evl_open_flags()]({{< relref "#evl_open_flags" >}}) have
been released, whether from the current process or any other process.

{{% argument flg %}}
The in-memory descriptor of the flag group to dismantle.
{{% /argument %}}

[evl_close_flags()]({{< relref "#evl_close_flags" >}}) returns zero
upon success. Otherwise, a negated error code is returned:

-EINVAL	      _flg_ does not represent a valid in-memory flag group
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

Closing a [statically initialized]({{< relref
"#EVL_FLAGS_INITIALIZER" >}}) flag group descriptor which has
never been used in wait or post operations always returns zero.

---

### Events pollable from an event flag group descriptor {#flags-poll-events}

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}})
interface can monitor the following events occurring on an event
flag group descriptor:

- _POLLIN_ and _POLLRDNORM_ are set whenever the flag group value is
  non-zero, which means that a subsequent attempt to read it by a call
  to [evl_wait_flags()]({{< relref "#evl_wait_flags" >}}),
  [evl_trywait_flags()]({{< relref "#evl_trywait_flags" >}}) or
  [evl_timedwait_flags()]({{< relref "#evl_timedwait_flags" >}})
  _might_ be successful without blocking (i.e. unless another thread
  sneaks in in the meantime and collects the pending flags).

- _POLLOUT_ and _POLLWRNORM_ are set whenever the flag group value is
  zero, which means that no flag is pending at the time of the
  call. As a result, polling for such status waits for all pending
  events to have been read by the receiving side.

---

{{<lastmodified>}}
