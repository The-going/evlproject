---
title: "Observable"
weight: 32
---

### Using the observer pattern for event-driven applications {#observer-pattern}

The EVL core provides a simple yet flexible support for implementing
the [observer design
pattern](https://en.wikipedia.org/wiki/Observer_pattern) in your
application, based on the [Observable]({{< relref
"core/_index.md#evl-core-elements" >}}) element.  The Observable
receives a stream of so-called _notices_, which may be events, state
changes or any data which fits within a 64bit word, delivering them as
_notifications_ (with additional meta-data) to subscribed threads
called _observers_, when they ask for it. Subscribers do not have to
be [attached]({{< relref
"core/user-api/thread/_index.md#evl_attach_thread" >}}) to the EVL
core, any thread can observe an Observable element. Whether the
threads which produce the events, the Observable and the observers
live in the same or different processes is transparent.

{{% mixedgrid-right src="/images/observable-bcast.png" %}}
#### Broadcast mode
By default, an Observable broadcasts a copy of every event it receives to
every subscribed observer:
{{% /mixedgrid-right %}}

{{% mixedgrid-right src="/images/observable-master.png" %}}
#### Master mode
In some cases, you may want to use an Observable to dispatch work
submitted as events to the set of observers which forms a pool of
worker threads.  Each message would be sent to a single worker, each
worker would be picked on a round-robin basis so as to implement a
naive load-balancing strategy.  Setting the Observable in the
so-called _master mode_ (see [EVL_CLONE_MASTER]({{< relref
"core/user-api/observable/_index.md#evl_create_observable" >}})) at
creation time enables this behavior:
{{% /mixedgrid-right %}}

### Creating an Observable

---

{{< proto evl_create_observable >}}
int evl_create_observable(int flags, const char *fmt, ...)
{{< /proto >}}

This call creates an Observable element, returning a file descriptor
representing the new object upon success. This is the generic call
form; for creating an Observable with common pre-defined settings, see
[evl_new_observable()}({{% relref "#evl_new_observable" %}}).

An Observable can operate either in [_broadcast_ or _master_ mode]({{<
relref "#observer-pattern" >}}):

- in broadcast mode, a copy of every event received by the Observable
  is sent to every [observer]({{< relref
  "core/user-api/thread/_index.md#evl_subscribe" >}}).

- in master mode, each message received by the Observable is sent to a
  single observer. The subscriber to send a message to is picked
  according to a simple round-robin strategy among all observers.

{{% argument flags %}}
A set of creation flags for the new element, defining its
[visibility]({{< relref "core/user-api/_index.md#element-visibility"
>}}) and operation mode:

  - `EVL_CLONE_PUBLIC` denotes a public element which is represented
    by a device file in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy, which
    makes it visible to other processes for sharing.
  
  - `EVL_CLONE_PRIVATE` denotes an element which is private to the
    calling process. No device file appears for it in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy.

  - `EVL_CLONE_MASTER` enables the [master mode]({{< relref
    "#observer-pattern" >}}) for the Observable.
  
  - `EVL_CLONE_NONBLOCK` sets the file descriptor of the new Observable in
    non-blocking I/O mode (`O_NONBLOCK`). By default, `O_NONBLOCK` is
    cleared for the file descriptor.
{{% /argument %}}

{{% argument fmt %}}
A [printf](http://man7.org/linux/man-pages/man3/printf.3.html)-like
format string to generate the Observable name. See this description of the
[naming convention]
({{< relref "core/user-api/_index.md#element-naming-convention" >}}).
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_create_observable()]({{< relref "#evl_create_observable" >}})
returns the file descriptor of the newly created Observable on
success. Otherwise, a negated error code is returned:

- -EEXIST	The generated name is conflicting with an existing Observable
  		name.

- -EINVAL	Either _flags_ is wrong, or the [generated name]({{< relref
  		"core/user-api/_index.md#element-naming-convention"
  		>}}) is badly formed.

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
		services in a way or another before creating an EVL Observable.

```
#include <evl/observable.h>

void create_new_observable(void)
{
	int efd;

	efd = evl_create_observable(EVL_CLONE_PUBLIC|EVL_CLONE_MASTER, "name_of_observable");
	/* skipping checks */
	
	return efd;
}
```
---

{{< proto evl_new_observable >}}
int evl_new_observable(const char *fmt, ...)
{{< /proto >}}

This call is a shorthand for creating a private observable operating
in [broadcast mode]({{< relref "#observer-pattern" >}}). It is
identical to calling:

```
	evl_create_observable(EVL_CLONE_PRIVATE, fmt, ...);
```

{{% notice info %}}
Note that if the [generated name] ({{< relref
"core/user-api/_index.md#element-naming-convention" >}}) starts with a
slash ('/') character, `EVL_CLONE_PRIVATE` would be automatically turned
into `EVL_CLONE_PUBLIC` internally.
{{% /notice %}}

---

### Working with an Observable {#working-with-an-observable}

Once an Observable is [created]({{< relref "#evl_create_observable"
>}}), using it entails the following steps, performed by either the
thread(s) issuing the stream of events to the Observable, or those
observing these events:

{{% mixedgrid-right src="/images/observable-subscribe.png" %}}
Observers need to subscribe to the Observable. Observers can subscribe
and unsubscribe at will, come and go freely during the Observable's
lifetime. Depending on its subscription flags, the observer may ask
for merging consecutive identical notices or receiving all of them
unfiltered (See [EVL_NOTIFY_ONCHANGE]({{< relref
"core/user-api/thread/_index.md#evl_subscribe" >}})).
{{% /mixedgrid-right %}}

{{% mixedgrid-right src="/images/observable-write.png" %}}
Event streamers can send notices to the Observable by calling
[evl_update_observable()]({{< relref "#evl_update_observable" >}}).
{{% /mixedgrid-right %}}

{{% mixedgrid-right src="/images/observable-read.png" %}}
Observers can read notifications from the Observable by calling
[evl_read_observable()]({{< relref "#evl_read_observable" >}}).
{{% /mixedgrid-right %}}

Once an Observable has become useless, you only need to close all the
file descriptors referring to it in order to release it, like for any
ordinary file.

---

{{< proto evl_update_observable >}}
int evl_update_observable(int efd, const struct evl_notice *ntc, int nr)
{{< /proto >}}

This call sends up to _nr_ notices starting at address _ntc_ in memory
to the Observable referred to by _efd_. This call never blocks the
caller. It may be used by any thread, including non-EVL ones.

A notice contains a _tag_, and an opaque _event_ value. It is defined by
the following C types:

```
union evl_value {
	int32_t val;
	int64_t lval;
	void *ptr;
};

struct evl_notice {
	uint32_t tag;
	union evl_value event;
};
```

All notices sent to the Observable should carry a valid issuer tag in
the `evl_notice.tag` field.  For applications, a valid tag is an
arbitrary value greater or equal to `EVL_NOTICE_USER` (lower tag
values are reserved to the core for [HM diag codes]({{< relref
"core/user-api/thread/_index.md#health-monitoring" >}})).
		
[evl_update_observable()]({{< relref "#evl_update_observable" >}})
returns the number of notices which have been successfully
queued. Each notice which have been successfully queued for
consumption by at least one observer counts for one in the return
value.  Zero is returned whenever no observer was subscribed to the
Observable at the time of the call, or no buffer space was available
for queuing any notification for any observer. Otherwise, a negated
error code is returned on error:

- -EBADF	_efd_ is not a valid file descriptor.

- -EPERM	_efd_ does not refer to an Observable element.
  		This may happen if _efd_ refers to a valid EVL thread
		which was not [created]({{< relref "#evl_attach_thread" >}})
		with the `EVL_CLONE_OBSERVABLE` flag.

- -EINVAL	Some notice has an invalid tag.

- -EFAULT	if _ntc_ points to invalid memory.

> Sending a notice to an observable
```
void send_notice(int ofd)
{
	struct evl_notice notice;
	int ret;

	notice.tag = EVL_NOTICE_USER;
	notice.event.val = 42;
	ret = evl_update_observable(ofd, &notice, 1);
	/* ret should be 1 on success. */
}
```

---

{{< proto evl_read_observable >}}
int evl_read_observable(int efd, struct evl_notification *nf, int nr)
{{< /proto >}}

This call receives up to _nr_ notifications starting at address _nf_
in memory from the Observable referred to by _efd_. It may only be
used by observers [subscribed]({{< relref
"core/user-api/thread/_index.md#evl_subscribe" >}}) to this particular
Observable. If `O_NONBLOCK` is clear for _efd_, the caller might block
until at least one notification arrives.

A notification contains the tag and the event value [sent by the
issuer]({{< relref "#evl_update_observable" >}}) of the corresponding
notice, plus some meta-data. A notification is defined by the
following C type:

```
union evl_value {
	int32_t val;
	int64_t lval;
	void *ptr;
};

struct evl_notification {
	uint32_t tag;
	uint32_t serial;
	int32_t issuer;
	union evl_value event;
	struct timespec date;
};
```

The meta-data is defined as follows:

- _serial_ is a monotonically increasing counter of notices sent by
  the Observable referred to by _efd_. In broadcast mode, this serial
  number is identical in every copy of a given original notice
  forwarded to the observers present.

- _issuer_ is the pid of the thread which issued the notice, zero if
  it was sent by the EVL core, such as with [HM diagnostics]({{< relref
  "core/user-api/thread/_index.md#health-monitoring" >}}).

- _date_ is a timestamp at receipt of the original notice, based on
  the built-in [EVL_CLOCK_MONOTONIC]({{< relref
  "core/user-api/clock/_index.md#builtin-clocks" >}}) clock.

[evl_read_observable()]({{< relref "#evl_read_observable" >}}) returns
the number of notifications which have been successfully copied to
_nf_. This count may be lower than _nr_. Otherwise, a negated error
code is returned on error:

- -EAGAIN	`O_NONBLOCK` is set for _efd_ and no notification is pending
  		at the time of the call.

- -EBADF	_efd_ is not a valid file descriptor.

- -EPERM	_efd_ does not refer to an Observable element.
  		This may happen if _efd_ refers to a valid EVL thread
		which was not [created]({{< relref "#evl_attach_thread" >}})
		with the `EVL_CLONE_OBSERVABLE` flag.

- -ENXIO	the calling thread is not [subscribed]({{< relref
  		"core/user-api/thread/_index.md#evl_subscribe" >}}) to
  		the Observable referred to by _efd_.

- -EFAULT	if _nf_ points to invalid memory.

> Receiving a notification from an observable
```
void receive_notification(int ofd)
{
	struct evl_notification notification;
	int ret;

	ret = evl_read_observable(ofd, &notification, 1);
	/* ret should be 1 on success. */
}
```

---

### Events pollable from an Observable {#observable-poll-events}

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}}) and
[poll(2)](http://man7.org/linux/man-pages/man2/poll.2.html) interfaces
can monitor the following events occurring on an Observable:

- `POLLIN` and `POLLRDNORM` are set whenever at least one notification
  is pending for the calling [observer thread]({{< relref
  "core/user-api/thread/_index.md#evl_subscribe" >}}). This means that
  a subsequent call to [evl_read_observable()]({{< relref
  "#evl_read_observable" >}}) by the same thread would return at least
  one valid notification immediately.

- `POLLOUT` and `POLLWRNORM` are set whenever at least one observer
  [subscribed]({{< relref
  "core/user-api/thread/_index.md#evl_subscribe" >}}) to the
  Observable has enough room in its backlog to receive at least one
  notice. This means that a subsequent call to
  [evl_update_observable()]({{< relref "#evl_read_observable" >}})
  would succeed in queuing at least one notice to one observer. In
  case multiple threads may update the Observable concurrently, which
  thread might succeed in doing so cannot be determined (typically,
  there would be no such guarantee for the caller of [evl_poll()]({{<
  relref "core/user-api/poll/_index.md" >}}) and
  [poll(2)](http://man7.org/linux/man-pages/man2/poll.2.html)).

In addition to these flags, `POLLERR` might be returned in case the
caller did not [subscribe]({{< relref
"core/user-api/thread/_index.md#evl_subscribe" >}}) to the Observable,
or some file descriptor from the polled set refer to an EVL thread
which was not [created]({{< relref "#evl_attach_thread" >}}) with the
`EVL_CLONE_OBSERVABLE` flag.

### Observing EVL threads {#observable-thread}

An EVL thread is in and of itself an Observable element. Observability
of a thread can be enabled by passing the `EVL_CLONE_OBSERVABLE` flag
when attaching a thread to the EVL core. In this case, the file
descriptor obtained from [evl_attach_thread()]({{< relref
"core/user-api/thread/_index.md#evl_attach_thread" >}}) may be
subsequently used in Observable-related operations. If the thread was
also made public ([EVL_CLONE_PUBLIC]({{< relref
"core/user-api/thread/_index.md#evl_attach_thread" >}})), then there
is a way for remote processes to monitor it via an access to [its
device]({{< relref "core/user-api/_index.md#evl-fs-hierarchy" >}}).

Threads can monitor events sent to an observable thread element by
[subscribing]({{< relref "core/user-api/thread/_index.md#evl_subscribe" >}})
to it. An observable thread typically relays [health monitoring]({{<
relref "core/user-api/thread/_index.md#health-monitoring" >}})
information to subscribers. An observable thread can also do
introspection, by subscribing to itself then reading the
HM diagnostics it has received.

---

{{<lastmodified>}}
