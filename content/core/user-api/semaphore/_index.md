---
title: "Semaphore"
weight: 20
---
### Synchronizing on a semaphore {#synchronizing-on-sema4}

EVL implements the classic Dijkstra semaphore construct, with an API
close to the [POSIX
specification](https://pubs.opengroup.org/onlinepubs/009695399/basedefs/semaphore.h.html)
for the basic operations.

### Semaphore services

---

{{< proto evl_create_sem >}}
int evl_create_sem(struct evl_sem *sem, int clockfd, init initval, int flags, const char *fmt, ...)
{{< /proto >}}

This call creates a semaphore, returning a file descriptor
representing the new object upon success. This is the generic call
form; for creating a semaphore with common pre-defined settings, see
[evl_new_sem()}({{% relref "#evl_new_sem" %}}).

{{% argument sem %}}
An in-memory semaphore descriptor is constructed by
[evl_create_sem()]({{< relref "#evl_create_sem" >}}),
which contains ancillary information other calls will need. _sem_
is a pointer to such descriptor of type `struct evl_sem`.
{{% /argument %}}

{{% argument clockfd %}}
Some semaphore-related calls are timed like
[evl_timedget_sem()]({{< relref "#evl_timedget_sem" >}})
which receives a timeout value. You can specify the [EVL clock]({{%
relref "core/user-api/clock/_index.md" %}}) this timeout refers to by
passing its file descriptor as _clockfd_. [Built-in EVL clocks]({{%
relref "core/user-api/clock/_index.md#builtin-clocks" %}}) are
accepted here.
{{% /argument %}}

{{% argument initval %}}
The initial value of the semaphore count.
{{% /argument %}}

{{% argument flags %}}
A set of creation flags for the new element, defining its
[visibility]({{< relref "core/user-api/_index.md#element-visibility"
>}}):

  - `EVL_CLONE_PUBLIC` denotes a public element which is represented
    by a device file in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy, which
    makes it visible to other processes for sharing.
  
  - `EVL_CLONE_PRIVATE` denotes an element which is private to the
    calling process. No device file appears for it in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy.

  - `EVL_CLONE_NONBLOCK` sets the file descriptor of the new semaphore in
    non-blocking I/O mode (`O_NONBLOCK`). By default, `O_NONBLOCK` is
    cleared for the file descriptor.
{{% /argument %}}

{{% argument fmt %}}
A [printf](http://man7.org/linux/man-pages/man3/printf.3.html)-like
format string to generate the semaphore name. See this description of the
[naming convention]
({{< relref "core/user-api/_index.md#element-naming-convention" >}}).
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_create_sem()]({{< relref "#evl_create_sem" >}}) returns the file
descriptor of the newly created semaphore on success. Otherwise, a
negated error code is returned:

- -EEXIST	The generated name is conflicting with an existing mutex,
		event, semaphore or flag group name.

- -EINVAL	Either _clockfd_ does not refer to a valid EVL clock, or the
  		generated semaphore name is badly formed, likely containing
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
		services in a way or another before creating an EVL semaphore.

```
#include <evl/sem.h>

static struct evl_sem sem;

void create_new_sem(void)
{
	int fd;

	fd = evl_create_sem(sem, EVL_CLOCK_MONOTONIC, 1, EVL_CLONE_PRIVATE, "name_of_semaphore");
	/* skipping checks */
	
	return fd;
}
```

---

{{< proto evl_new_sem >}}
int evl_new_sem(struct evl_sem *sem, const char *fmt, ...)
{{< /proto >}}

This call is a shorthand for creating a zero-initialized private
semaphore, timed on the built-in [EVL monotonic clock]({{% relref
"core/user-api/clock/_index.md#builtin-clocks" %}}). It is identical
to calling:

```
	evl_create_sem(sem, EVL_CLOCK_MONOTONIC, 0, EVL_CLONE_PRIVATE, fmt, ...);
```

{{% notice info %}}
Note that if the [generated name] ({{< relref
"core/user-api/_index.md#element-naming-convention" >}}) starts with a
slash ('/') character, `EVL_CLONE_PRIVATE` would be automatically turned
into `EVL_CLONE_PUBLIC` internally.
{{% /notice %}}

---

{{< proto EVL_SEM_INITIALIZER >}}
EVL_SEM_INITIALIZER((const char *) name, (int) clockfd, (int) initval, (int) flags)
{{< /proto >}}

The static initializer you can use with semaphores. All arguments to
this macro refer to their counterpart in the call to
[evl_create_sem()]({{< relref "#evl_create_sem" >}}).

```
struct evl_sem sem = EVL_SEM_INITIALIZER("name_of_semaphore", EVL_CLOCK_MONOTONIC, 1, EVL_CLONE_PUBLIC);
```

---

{{< proto evl_open_sem >}}
int evl_open_sem(struct evl_sem *sem, const char *fmt, ...)
{{< /proto >}}

You can open an existing semaphore, possibly from a different
process, by calling [evl_open_sem()]({{< relref "#evl_open_sem" >}}).

{{% argument sem %}}
An in-memory semaphore descriptor is constructed by
[evl_open_sem()]({{< relref "#evl_open_sem" >}}),
which contains ancillary information other calls will need. _sem_ is a
pointer to such descriptor of type `struct evl_sem`. The information
is retrieved from the existing semaphore which was opened.
{{% /argument %}}

{{% argument fmt %}}
A printf-like format string to generate the name of the semaphore to
open. This name must exist in the EVL device file hierarchy at
[/dev/evl/monitor]({{< relref
"core/user-api/_index.md#evl-fs-hierarchy" >}}).
See this description of the [naming convention]({{<
relref "core/user-api/_index.md#element-naming-convention" >}}).
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_open_sem()]({{< relref "#evl_open_sem" >}}) returns the file
descriptor referring to the opened semaphore on success, Otherwise, a
negated error code is returned:

- -EINVAL	The name refers to an existing object, but not to a semaphore.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -ENOMEM	No memory available.

---

{{< proto evl_get_sem >}}
int evl_get_sem(struct evl_sem *sem)
{{< /proto >}}

This service [decrements a semaphore]({{% relref
"#synchronizing-on-sema4" %}}) by one.  If the resulting semaphore
value is negative, the caller is put to sleep by the core until a
subsequent call to [evl_put_sem()]({{< relref "#evl_put_sem" >}})
eventually releases it. Otherwise,
the caller returns immediately.  Waiters are queued by order of
[scheduling priority]({{< relref "core/user-api/scheduling/_index.md"
>}}).

{{% argument sem %}}
The in-memory semaphore descriptor constructed by
either [evl_create_sem()]({{< relref "#evl_create_sem" >}}) or
[evl_open_sem()]({{< relref "#evl_open_sem" >}}), or statically built
with [EVL_SEM_INITIALIZER]({{% relref "#EVL_SEM_INITIALIZER"
%}}). In the latter case, an implicit call to
[evl_create_sem()]({{< relref "#evl_create_sem" >}}) for
_sem_ is issued before a get operation is attempted, which may trigger
a transition to the in-band execution mode for the caller.
{{% /argument %}}

[evl_get_sem()]({{< relref "#evl_get_sem" >}}) returns zero on
success. Otherwise, a negated error code may be returned if:

-EINVAL	      _sem_ does not represent a valid in-memory semaphore
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

If _sem_ was statically initialized with [EVL_SEM_INITIALIZER]({{%
relref "#EVL_SEM_INITIALIZER" %}}), then any error returned by
[evl_create_sem()]({{% relref "#evl_create_sem" %}}) may be passed
back to the caller in case the implicit initialization call fails.

---

{{< proto evl_timedget_sem >}}
int evl_timedget_sem(struct evl_sem *sem, const struct timespec *timeout)
{{< /proto >}}

This call is a variant of [evl_get_sem()]({{% relref "#evl_get_sem"
%}}) which allows specifying a timeout on the get operation, so that
the caller is unblocked after a specified delay sleeping without being
unblocked by a subsequent call to [evl_put_sem()]({{% relref
"#evl_put_sem" %}}).

{{% argument sem %}}
The in-memory semaphore descriptor constructed by
either [evl_create_sem()]({{< relref "#evl_create_sem" >}}) or
[evl_open_sem()]({{< relref "#evl_open_sem" >}}), or statically built
with [EVL_SEM_INITIALIZER]({{% relref "#EVL_SEM_INITIALIZER"
%}}). In the latter case, an implicit call to
[evl_create_sem()]({{< relref "#evl_create_sem" >}}) is
issued for _sem_ before a get operation is attempted, which may
trigger a transition to the in-band execution mode for the caller.
{{% /argument %}}

{{% argument timeout %}}
A time limit to wait for the caller to be unblocked before
the call returns on error. The clock mentioned in the call to
[evl_create_sem()]({{% relref "#evl_create_sem" %}}) will be used for
tracking the elapsed time.
{{% /argument %}}

The possible return values include any status from
[evl_get_sem()]({{% relref "#evl_get_sem" %}}), plus:

-ETIMEDOUT	   The timeout fired, after the amount of time specified by
		   _timeout_.

---

{{< proto evl_put_sem >}}
int evl_put_sem(struct evl_sem *sem)
{{< /proto >}}

This call posts a semaphore by incrementing its count by one. If a
thread is sleeping on the semaphore as a result of a previous call to
[evl_get_sem()]({{% relref "#evl_get_sem" %}}) or
[evl_timedget_sem()]({{% relref "#evl_timedget_sem" %}}), the thread
heading the wait queue is unblocked.

{{% argument sem %}}
The in-memory semaphore descriptor constructed by either
[evl_create_sem()]({{< relref "#evl_create_sem" >}})
or [evl_open_sem()]({{< relref "#evl_open_sem" >}}), or statically built with
[EVL_SEM_INITIALIZER]({{% relref "#EVL_SEM_INITIALIZER" %}}). In
the latter case, an implicit call to
[evl_create_sem()]({{< relref "#evl_create_sem" >}}) for _sem_ is
issued before the semaphore is posted, which may trigger a transition to
the in-band execution mode for the caller.
{{% /argument %}}

[evl_put_sem()]({{< relref "#evl_put_sem" >}}) returns zero upon
success. Otherwise, a negated error code is returned:

-EINVAL	      _sem_ does not represent a valid in-memory semaphore
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

If _sem_ was statically initialized with [EVL_SEM_INITIALIZER]({{%
relref "#EVL_SEM_INITIALIZER" %}}) but not passed to any
semaphore-related call yet, then any error status returned by
[evl_create_sem()]({{% relref "#evl_create_sem" %}}) may be passed
back to the caller in case the implicit initialization call fails.

---

{{< proto evl_tryget_sem >}}
int evl_tryget_sem(struct evl_sem *sem, int *r_bits)
{{< /proto >}}

This call attempts to decrement the semaphore provided the result does
not lead to a negative count.

{{% argument sem %}}
The in-memory semaphore descriptor constructed by
either [evl_create_sem()]({{< relref "#evl_create_sem" >}}) or
[evl_open_sem()]({{< relref "#evl_open_sem" >}}), or statically built
with [EVL_SEM_INITIALIZER]({{% relref "#EVL_SEM_INITIALIZER"
%}}). In the latter case, an implicit call to
[evl_create_sem()]({{< relref "#evl_create_sem" >}}) for
_sem_ is issued before a wait is attempted, which may trigger a transition
to the in-band execution mode for the caller.
{{% /argument %}}

[evl_tryget_sem()]({{< relref "#evl_tryget_sem" >}}) returns zero on
success. Otherwise, a negated error code may be returned if:

-EAGAIN       _sem_ count value is zero or negative at the time of the call.

-EINVAL	      _sem_ does not represent a valid in-memory semaphore
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

If _sem_ was statically initialized with [EVL_SEM_INITIALIZER]({{%
relref "#EVL_SEM_INITIALIZER" %}}), then any error returned by
[evl_create_sem()]({{% relref "#evl_create_sem" %}}) may be passed
back to the caller in case the implicit initialization call fails.

---

{{< proto evl_peek_sem >}}
int evl_peek_sem(struct evl_sem *sem, int *r_val)
{{< /proto >}}

This call returns the count value of the semaphore. If a negative
count is returned in *_r\_val_, its absolute value can be interpreted
as the number of waiters sleeping on the semaphore's wait queue (at
the time of the call). A zero or positive value means that the
semaphore is not contended.

{{% argument sem %}}
The in-memory semaphore descriptor constructed by either
[evl_create_sem()]({{< relref "#evl_create_sem" >}})
or [evl_open_sem()]({{< relref "#evl_open_sem"
>}}), or statically built with [EVL_SEM_INITIALIZER]({{%
relref "#EVL_SEM_INITIALIZER" %}}). In the latter case, the
semaphore becomes valid for a call to [evl_peek_sem()]({{< relref
"#evl_peek_sem" >}}) only after a put or [try]get operation was issued
for it.
{{% /argument %}}

{{% argument r_val %}}
The address of an integer which contains the semaphore value on
successful return from the call.
{{% /argument %}}

[evl_peek_sem()]({{< relref "#evl_peek_sem" >}}) returns zero on
success along with the current semaphore count.  Otherwise, a negated
error code may be returned if:

-EINVAL	      _sem_ does not represent a valid in-memory semaphore
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

---

{{< proto evl_close_sem >}}
int evl_close_sem(struct evl_sem *sem)
{{< /proto >}}

You can use [evl_close_sem()]({{< relref "#evl_close_sem" >}}) to
dispose of an EVL semaphore, releasing the associated file descriptor,
at which point _sem_ will not be valid for any subsequent operation
from the current process. However, this semaphore is kept alive in the
EVL core until all file descriptors opened on it by call(s) to
[evl_open_sem()]({{< relref "#evl_open_sem" >}}) have been released,
whether from the current process or any other process.

{{% argument sem %}}
The in-memory descriptor of the semaphore to dismantle.
{{% /argument %}}

[evl_close_sem()]({{< relref "#evl_close_sem" >}}) returns zero upon
success. Otherwise, a negated error code is returned:

-EINVAL	      _sem_ does not represent a valid in-memory semaphore
	      descriptor. If that pointer is out of the caller's
	      address space or points to read-only memory, the
	      caller bluntly gets a memory access exception.

Closing a [statically initialized]({{< relref "#EVL_SEM_INITIALIZER"
>}}) semaphore descriptor which has never been used in get or put
operations always returns zero.

---

### Events pollable from a semaphore file descriptor {#sema4-poll-events}

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}})
interface can monitor the following events occurring on a semaphore
file descriptor:

- _POLLIN_ and _POLLRDNORM_ are set whenever the semaphore count is
  strictly positive, which means that a subsequent attempt to deplete
  it by a call to [evl_get_sem()]({{< relref "#evl_get_sem" >}}),
  [evl_tryget_sem()]({{< relref "#evl_tryget_sem" >}}) or
  [evl_timedget_sem()]({{< relref "#evl_timedget_sem" >}}) _might_ be
  successful without blocking (i.e. unless another thread sneaks in
  in the meantime and fully depletes the semaphore).

---

{{<lastmodified>}}
