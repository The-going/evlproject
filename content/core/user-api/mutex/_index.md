---
title: "Mutex"
weight: 10
---

### Serializing threads with mutexes {#serializing-with-mutex}

EVL provides common mutexes for serializing thread access to a shared
resource from [out-of-band context]({{< relref
"dovetail/pipeline/_index.md" >}}), with semantics close to the [POSIX
specification](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/pthread.h.html)
for the basic operations.

### Mutex services

---

{{< proto evl_new_mutex_any >}}
int evl_new_mutex_any(struct evl_mutex *mutex, int type, int clockfd, unsigned int ceiling, const char *fmt, ...)
{{< /proto >}}

This call creates a mutex, returning a file descriptor representing
the new object upon success. This is the generic call form; for
creating a mutex with common pre-defined settings, see
[evl_new_mutex()}({{% relref "#evl_new_mutex" %}}).

{{% argument mutex %}}
An in-memory mutex descriptor is constructed by `evl_new_mutex_any()`,
which contains ancillary information other calls will need. _mutex_
is a pointer to such descriptor of type `struct evl_mutex`.
{{% /argument %}}

{{% argument type %}}
The basic type of the mutex: either EVL_MUTEX_NORMAL for a normal,
non-recursive mutex, or EVL_MUTEX_RECURSIVE for a recursive mutex a
thread may lock multiple times in a nested way.
{{% /argument %}}

{{% argument clockfd %}}
Some mutex-related calls are timed like `evl_timedlock_mutex()`
which receives a timeout value. You can specify the [EVL clock]({{%
relref "core/user-api/clock/_index.md" %}}) this timeout refers to by
passing its file descriptor as _clockfd_. [Built-in EVL clocks]({{%
relref "core/user-api/clock/_index.md#builtin-clocks" %}}) are
accepted here.
{{% /argument %}}

{{% argument ceiling %}}
When non-zero, EVL interprets this value as the priority ceiling of
the new mutex. In this case, the behavior is similar to POSIX's
[PRIO_PROTECT
protocol](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_mutexattr_setprotocol.html).
The priority must be in the [1-99] range, so that priority ceiling can
apply to any [scheduling policy]({{< relref "core/user-api/scheduling"
>}}) EVL supports.

When zero, EVL applies
a priority inheritance protocol such as described for POSIX's
[PRIO_INHERIT
protocol](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_mutexattr_setprotocol.html).
{{% /argument %}}

{{% notice note %}}
For priority protection/ceiling, EVL actually changes the priority of
the lock owner to the ceiling value only when/if the rescheduling
procedure is invoked while such thread is current on the CPU. Since
mutex-protected critical sections should be short and the lock owner
is unlikely to schedule out while holding such lock, this reduces the
overhead of dealing with priority protection, whilst providing the
same guarantees compared to changing the priority immediately upon
acquiring that lock.
{{% /notice %}}

{{% argument fmt %}}
A printf-like format string to generate the mutex name. A common way
of generating unique names is to add the calling process's
_pid_ somewhere into the format string as illustrated in the
example. The generated name is used to form a last part of a pathname,
referring to the new [monitor element]({{< relref "core/_index.md" >}})
device underpinning the mutex in the file system. So this name must
contain only valid characters in this context, excluding slashes.
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

`evl_new_mutex_any()` returns the file descriptor of the newly created
mutex on success. Otherwise, a negated error code is returned:

- -EEXIST	The generated name is conflicting with an existing mutex,
		event, semaphore or flag group name.

- -EINVAL	Either _clockfd_ does not refer to a valid EVL clock, or the
  		generated mutex name is badly formed, likely containing
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
		services in a way or another before creating an EVL mutex.

```
#include <evl/mutex.h>

static struct evl_mutex mutex;

void create_new_mutex(void)
{
	int fd;

	/* Create a non-recursive mutex with priority inheritance enabled. */
	fd = evl_new_mutex_any(mutex, EVL_MUTEX_NORMAL, EVL_CLOCK_MONOTONIC, 0, "name_of_mutex");
	/* skipping checks */
	
	return fd;
}
```

---

{{< proto evl_new_mutex >}}
int evl_new_mutex(struct evl_mutex *mutex, const char *fmt, ...)
{{< /proto >}}

This call creates a non-recursive mutex enabling the priority
inheritance protocol, timed on the built-in [EVL monotonic clock]({{%
relref "core/user-api/clock/_index.md#builtin-clocks" %}}). It is
identical to calling:

```
evl_new_mutex_any(mutex, EVL_MUTEX_NORMAL, EVL_CLOCK_MONOTONIC, 0, fmt, ...);
```

---

{{< proto EVL_MUTEX_ANY_INITIALIZER >}}
EVL_MUTEX_ANY_INITIALIZER(type, name, clockfd, ceiling)
{{< /proto >}}

The static initializer you can use with mutexes. This is the
generic form; for initializing a mutex with common pre-defined
settings, see [EVL_MUTEX_INITIALIZER]({{% relref "#EVL_MUTEX_INITIALIZER"
%}}).

{{% argument type %}}
The basic type of the mutex as described for [evl_new_mutex_any()]({{%
relref "#evl_new_mutex_any" %}}).
{{% /argument %}}

{{% argument name %}}
A name which is used to form a last part of a pathname, referring to
the new [monitor element]({{< relref "core/_index.md" >}}) device
underpinning the mutex in the file system. So this name must contain
only valid characters in this context, excluding slashes.
{{% /argument %}}

{{% argument clockfd %}}
The reference clock for timed operations on the mutex as described for
[evl_new_mutex_any()]({{% relref "#evl_new_mutex_any" %}}).
{{% /argument %}}

{{% argument ceiling %}}
The priority ceiling for the mutex  as described for
[evl_new_mutex_any()]({{% relref "#evl_new_mutex_any" %}}).
{{% /argument %}}

```
/* Create a recursive mutex with priority ceiling enabled (prio=90). */
struct evl_mutex mutex = EVL_MUTEX_ANY_INITIALIZER(EVL_MUTEX_RECURSIVE, "name_of_mutex", EVL_CLOCK_MONOTONIC, 90);
```

---

{{< proto EVL_MUTEX_INITIALIZER >}}
EVL_MUTEX_INITIALIZER(name)
{{< /proto >}}

This is a short-hand initializer specifying a non-recursive mutex,
enabling the priority inheritance protocol and timed by the built-in
[EVL monotonic clock]({{% relref
"core/user-api/clock/_index.md#builtin-clocks" %}}). It is identical
to writing:

```
EVL_MUTEX_ANY_INITIALIZER(EVL_MUTEX_NORMAL, name, EVL_CLOCK_MONOTONIC, 0);
```

---

{{< proto evl_open_mutex >}}
int evl_open_mutex(struct evl_mutex *mutex, const char *fmt, ...)
{{< /proto >}}

You can open an existing mutex, possibly from a different process, by
calling `evl_open_mutex()`.

{{% argument mutex %}}
An in-memory mutex descriptor is constructed by `evl_open_mutex()`,
which contains ancillary information other calls will need. _mutex_ is a
pointer to such descriptor of type `struct evl_mutex`. The information
is retrieved from the existing mutex which was opened.
{{% /argument %}}

{{% argument fmt %}}
A printf-like format string to generate the name of the mutex to
open. This name must exist in the EVL device hierachy at
_/dev/evl/monitor/_.
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

`evl_open_mutex()` returns the file descriptor referring to the opened
mutex on success, Otherwise, a negated error code is returned:

- -EINVAL	The name refers to an existing object, but not to a mutex.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

---

{{< proto evl_lock_mutex >}}
int evl_lock_mutex(struct evl_mutex *mutex)
{{< /proto >}}

The current thread may lock a mutex by calling this service, which
guarantees exclusive access to the resource it protects on success,
until it eventually releases _mutex_.

If _mutex_ is unlocked, or it is of recursive type and the current
thread already owns it on entry to the call, the lock nesting count is
incremented by one then `evl_lock_mutex()` returns immediately with a
success status.

Otherwise, the caller blocks until the current owner releases it by a
call to [evl_unlock_mutex()]({{% relref "#evl_unlock_mutex" %}}). If
multiple threads wait for acquiring the lock, the one with the
[highest scheduling priority]({{< relref "core/user-api/scheduling"
>}}) which has been waiting for the longest time is served first.

{{% argument mutex %}}
The in-memory mutex descriptor constructed by
either `evl_new_mutex[_any]()` or `evl_open_mutex()`, or statically built
with [EVL_MUTEX\[_ANY\]_INITIALIZER]({{% relref "#EVL_MUTEX_ANY_INITIALIZER"
%}}). In the latter case, an implicit call to `evl_new_mutex_any()` is
issued for _mutex_ before a locking operation is attempted, which may
trigger a transition to the in-band execution mode for the caller.
{{% /argument %}}

`evl_lock_mutex()` returns zero on success. Otherwise, a negated error
code may be returned if:

-EAGAIN       _mutex_ is recursive and the lock nesting count would
	      exceed 2^31.

-EINVAL	      _mutex_ does not represent a valid in-memory mutex
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

If _mutex_ was statically initialized with [EVL_MUTEX\[_ANY\]_INITIALIZER]({{%
relref "#EVL_MUTEX_ANY_INITIALIZER" %}}), then any error returned by
[evl_new_mutex_any()]({{% relref "#evl_new_mutex_any" %}}) may be passed
back to the caller in case the implicit initialization call fails.

---

{{< proto evl_timedlock_mutex >}}
int evl_timedlock_mutex(struct evl_mutex *mutex)
{{< /proto >}}

This call is a variant of [evl_lock_mutex()]({{% relref
"#evl_lock_mutex" %}}) which allows specifying a timeout on the
locking operation, so that the caller is unblocked after a specified
delay without being able to acquire the lock.

{{% argument mutex %}}
The in-memory mutex descriptor constructed by
either `evl_new_mutex[_any]()` or `evl_open_mutex()`, or statically built
with [EVL_MUTEX\[_ANY\]_INITIALIZER]({{% relref "#EVL_MUTEX_ANY_INITIALIZER"
%}}). In the latter case, an implicit call to `evl_new_mutex_any()` is
issued for _mutex_ before a locking operation is attempted, which may
trigger a transition to the in-band execution mode for the caller.
{{% /argument %}}

{{% argument timeout %}}
A time limit to wait for the caller to be unblocked before
the call returns on error. The clock mentioned in the call to
[evl_new_mutex_any()]({{% relref "#evl_new_mutex_any" %}}) will be used for
tracking the elapsed time.
{{% /argument %}}

The possible return values include any status from
[evl_lock_mutex()]({{% relref "#evl_lock_mutex" %}}), plus:

-ETIMEDOUT	   The timeout fired, after the amount of time specified by
		   _timeout_.

---

{{< proto evl_trylock_mutex >}}
int evl_timedlock_mutex(struct evl_mutex *mutex)
{{< /proto >}}

This call attempts to lock the mutex like [evl_lock_mutex()]({{%
relref "#evl_lock_mutex" %}}) would do, but returns immediately both
on success or failure to do so, without waiting for the lock to be
available in the latter case.

{{% argument mutex %}}
The in-memory mutex descriptor constructed by
either `evl_new_mutex[_any]()` or `evl_open_mutex()`, or statically built
with [EVL_MUTEX\[_ANY\]_INITIALIZER]({{% relref "#EVL_MUTEX_ANY_INITIALIZER"
%}}). In the latter case, an implicit call to `evl_new_mutex_any()` for
_mutex_ is issued before a wait is attempted, which may trigger a transition
to the in-band execution mode for the caller.
{{% /argument %}}

`evl_trylock_mutex()` returns zero on success. Otherwise, a negated error
code may be returned if:

-EAGAIN       _mutex_ is already locked by another thread.

-EINVAL	      _mutex_ does not represent a valid in-memory mutex
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

If _mutex_ was statically initialized with [EVL_MUTEX\[_ANY\]_INITIALIZER]({{%
relref "#EVL_MUTEX_ANY_INITIALIZER" %}}), then any error returned by
[evl_new_mutex_any()]({{% relref "#evl_new_mutex_any" %}}) may be passed
back to the caller in case the implicit initialization call fails.

---

{{< proto evl_unlock_mutex >}}
int evl_unlock_mutex(struct evl_mutex *mutex)
{{< /proto >}}

This routine releases a mutex previously acquired by
[evl_lock_mutex()]({{% relref "#evl_lock_mutex" %}}),
[evl_trylock_mutex()]({{% relref "#evl_trylock_mutex" %}}) or
[evl_timedlock_mutex()]({{% relref "#evl_timedlock_mutex" %}}) by the
current EVL thread.

{{% notice note %}}
Only the thread which acquired an EVL mutex may release it.
{{% /notice %}}

{{% argument mutex %}}
The in-memory mutex descriptor constructed by either
`evl_new_mutex[_any]()` or `evl_open_mutex()`.
{{% /argument %}}

If _mutex_ is recursive, the lock nesting count is decremented by one
until it reaches zero, at which point the lock is actually released.

`evl_unlock_mutex()` returns zero on success. Otherwise, a negated error
code may be returned if:

-EPERM	      The caller does not own the mutex.

-EINVAL	      _mutex_ does not represent a valid in-memory mutex
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

---

{{< proto evl_set_mutex_ceiling >}}
int evl_set_mutex_ceiling(struct evl_mutex *mutex, unsigned int ceiling)
{{< /proto >}}

Change the priority ceiling value for a mutex. If the mutex is
currently owned by a thread, the change may not apply immediately,
depending on whether the priority ceiling was committed for that
thread already (see [this note]({{% relref "#evl_new_mutex_any" %}})).

{{% argument mutex %}}
The in-memory mutex descriptor constructed by either
`evl_new_mutex[_any]()` or `evl_open_mutex()`.
{{% /argument %}}

{{% argument ceiling %}}
The new priority ceiling in the [1-99] range.
{{% /argument %}}

`evl_set_mutex_ceiling()` returns zero on success. Otherwise, a negated error
code may be returned if:

-EINVAL	      _mutex_ does not enable priority protection.

-EINVAL	      _ceiling_ is out of range.

-EINVAL	      _mutex_ does not represent a valid in-memory mutex
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.
---

{{< proto evl_get_mutex_ceiling >}}
int evl_get_mutex_ceiling(struct evl_mutex *mutex)
{{< /proto >}}

Retrieve the current priority ceiling value for _mutex_. This call
also succeeds for a mutex which does not enable priority protection
(but priority inheritance instead), returning zero in such a case.

{{% argument mutex %}}
The in-memory mutex descriptor constructed by either
`evl_new_mutex[_any]()` or `evl_open_mutex()`.
{{% /argument %}}

`evl_get_mutex_ceiling()` returns the current priority ceiling value
on success. Otherwise, a negated error code may be returned if:

-EINVAL	      _mutex_ does not represent a valid in-memory mutex
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.
---

{{< proto evl_close_mutex >}}
int evl_close_mutex(struct evl_mutex *mutex)
{{< /proto >}}

You can use `evl_close_mutex()` to dispose of an EVL mutex,
releasing the associated file descriptor, at which point _mutex_ will
not be valid for any subsequent operation from the current
process. However, this mutex is kept alive in the EVL core until
all file descriptors opened on it by call(s) to [evl_open_mutex()]({{<
relref "#evl_open_mutex" >}}) have been released, whether from the
current process or any other process.

{{% argument mutex %}}
The in-memory descriptor of the mutex to dismantle.
{{% /argument %}}

`evl_close_mutex()` returns zero upon success. Otherwise, a negated
error code is returned:

-EINVAL	      _mutex_ does not represent a valid in-memory mutex
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

Closing a [statically initialized]({{< relref
"#EVL_MUTEX_ANY_INITIALIZER" >}}) mutex descriptor which has never
been locked always returns zero.
