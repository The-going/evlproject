---
title: "Polling file descriptors"
weight: 45
---

### Waiting for events on file descriptors

Every [EVL element]({{%relref "core/_index.md#evl-core-elements" %}})
is represented by a device file accessible in the [/dev/evl]({{<
relref "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy,
and requests can be sent to any of those elements by writing to common
file descriptors obtained on the corresponding files. Conversely,
information is returned to the caller by reading from those file
descriptors. EVL applications can poll for out-of-band events
occurring on a set of file descriptors opened on elements, just like
in-band applications can use
[poll(2)](http://man7.org/linux/man-pages/man2/poll.2.html) with other
files.

To access the out-of-band polling services, an application first needs
to obtain a file descriptor on a new _polling set_ by a call to
[evl_poll()]({{% relref "#evl_new_poll" %}}). Such descriptor is an
access point for registering other file descriptors to monitor, then
[sensing events]({{% relref "#evl_poll" %}}) occurring on them.
[evl_poll()]({{% relref "#evl_poll" %}}) or [evl_timedpoll()]({{%
relref "#evl_timedpoll" %}}) fill an array of structures indicating
which events have been received on each notified file descriptor. The
type of this structure is as follows:

```
struct evl_poll_event {
	__u32 fd;
	__u32 events;
};
```

Applications can wait for any event which
[poll(2)](http://man7.org/linux/man-pages/man2/poll.2.html) can
monitor such as `POLLIN`, `POLLOUT`, `POLLRDNORM`, `POLLWRNORM` and so
on. Which event can occur on a given file descriptor is defined by the
out-of-band driver managing the file it refers to.

<table style="width:10%; text-align: center;">
  <tr>
    <th>Pollable elements</th>
  </tr>
  <tr>
    <td><a href="/core/user-api/semaphore/index.html#sema4-poll-events" target="_blank">Semaphore</a></td>
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#flags-poll-events" target="_blank">Event flag group</a></td>
  </tr>
  <tr>
    <td><a href="/core/user-api/timer/index.html#timer-poll-events" target="_blank">Timer</a></td>
  </tr>
  <tr>
    <td><a href="/core/user-api/proxy/index.html#proxy-poll-events" target="_blank">Proxy</a></td>
  </tr>
  <tr>
    <td><a href="/core/user-api/xbuf/index.html#xbuf-poll-events" target="_blank">Cross-buffer</a></td>
  </tr>
</table>

### Polling services

{{< proto evl_new_poll >}}
int evl_new_poll(void)
{{< /proto >}}

This call creates a polling set, returning a file descriptor
representing the new object upon success. There is no arbitrary limit
on the number of polling sets an application can create, which is only
limited to the available system resources.

`evl_new_poll()` returns the file descriptor of the newly created
polling set on success. Otherwise, a negated error code is returned:

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
  		before creating an EVL polling set.

---

{{< proto evl_add_pollfd >}}
int evl_add_pollfd(int efd, int newfd, unsigned int events)
{{< /proto >}}

Adds a file descriptor to a polling set. This service registers the
_newfd_ file descriptor for being polled for _events_ by the _efd_
polling set.

{{% argument efd %}}
The file descriptor of a polling set obtained by a previous call to
[evl_new_poll()]({{% relref "#evl_new_poll" %}}).
{{% /argument %}}

{{% argument newfd %}}
A file descriptor to monitor for events. Once _newfd_ is added to the
polling set, it becomes a source of events which is monitored when
reading from _efd_. You can add as many file descriptor as you need to
a polling set in order to perform synchronous I/O multiplexing over
multiple data sources or sinks.
{{% /argument %}}

{{% argument events %}}
A bitmask representing the set of events to monitor for _newfd_. These
events are common to the in-band
[poll(2)](http://man7.org/linux/man-pages/man2/poll.2.html)
interface. `POLLERR` and `POLLHUP` are always implicitly polled, even
if they are not mentioned in _events_. `POLLNVAL` cannot be awaited
for. Which event is meaningful in the context of the call depends on
the driver managing the file, which defines the logic for raising
them.  For instance, if _newfd_ refers to an [EVL semaphore]({{%
relref "core/user-api/semaphore/_index.md#sema4-poll-events" %}}), you
could monitor `POLLIN` and `POLLRDNORM`, no other event would occur
for this type of file.
{{% /argument %}}

`evl_add_pollfd()` returns zero on success. Otherwise, a negated error
code is returned:

- -EBADF	 _efd_ is not a valid file descriptor referring to an EVL
  		 polling set.

- -EBADF	 _newfd_ is not a valid file descriptor to an EVL file. Note
  		 that you cannot register a file descriptor referring to a
		 common file (i.e. managed by an in-band driver) into an
		 EVL polling set. An EVL file is created by [out-of-band drivers]
		 ({{% relref "core/user-api/io/_index.md" %}}) for referring
		 to resources which can be accessed from the [out-of-band execution]
		 ({{% relref "dovetail/altsched.md" %}}) stage.

- -ELOOP	 Adding _newfd_ to the polling set would lead to a cyclic
  		 dependency (such as _newfd_ referring to the polling set).

- -ENOMEM	 No memory available.

---

{{< proto evl_del_pollfd >}}
int evl_del_pollfd(int efd, int delfd)
{{< /proto >}}

Removes a file descriptor from a polling set. As a result, reading
this polling set does not monitor _delfd_ anymore. This change applies
to subsequent calls to [evl_poll()]({{% relref "#evl_poll" %}}) or
[evl_timedpoll()]({{% relref "#evl_timedpoll" %}}) for such set;
ongoing waits are not affected.

{{% argument efd %}}
The file descriptor of a polling set obtained by a previous call to
[evl_new_poll()]({{% relref "#evl_new_poll" %}}).
{{% /argument %}}

{{% argument delfd %}}
The file descriptor to stop monitoring for events. This descriptor
should have been registered by an earlier call to
[evl_add_pollfd()]({{% relref "#evl_add_pollfd" %}}).
{{< /proto >}}

`evl_del_pollfd()` returns zero on success. Otherwise, a negated error
code is returned:

- -EBADF	 _efd_ is not a valid file descriptor referring to an EVL
  		 polling set.

- -ENOENT	 _delfd_ is not a file descriptor present into the polling
  		 set referred to by _efd_.

---

{{< proto evl_mod_pollfd >}}
int evl_mod_pollfd(int efd, int modfd, unsigned int events)
{{< /proto >}}

Modifies the _events_ being monitored for a file descriptor already
registered in a polling set. This change applies to subsequent calls
to [evl_poll()]({{% relref "#evl_poll" %}}) or [evl_timedpoll()]({{%
relref "#evl_timedpoll" %}}) for such set; ongoing waits are not
affected.

{{% argument efd %}}
The file descriptor of a polling set obtained by a previous call to
[evl_new_poll()]({{% relref "#evl_new_poll" %}}).
{{% /argument %}}

{{% argument modfd %}}
The file descriptor to update the event mask for. This descriptor
should have been registered by an earlier call to
[evl_add_pollfd()]({{% relref "#evl_add_pollfd" %}}).
{{< /proto >}}

`evl_mod_pollfd()` returns zero on success. Otherwise, a negated error
code is returned:

- -EBADF	 _efd_ is not a valid file descriptor referring to an EVL
  		 polling set.

- -ENOENT	 _pollfd_ is not a file descriptor present into the polling
  		 set referred to by _efd_.

---

{{< proto evl_poll >}}
int evl_poll(int efd, struct evl_poll_event *pollset, int nrset)
{{< /proto >}}

Waits for events on the file descriptors present in the polling set
referred to by _efd_.

{{% argument efd %}}
The file descriptor of a polling set obtained by a previous call to
[evl_new_poll()]({{% relref "#evl_new_poll" %}}).
{{% /argument %}}

{{% argument pollset %}}
The start of an array of structures of type `evl_poll_event`
containing the set of file descriptors which have received at least
one event when the call returns.
{{% /argument %}}

{{% argument nrset %}}
The number of structures in the _pollset_ array. If zero,
[evl_poll()]({{% relref "#evl_new_poll" %}}) returns immediately with
a zero count.
{{% /argument %}}

On success, `evl_poll()` returns the count of file descriptors from
the polling set for which an event is pending, each of them will
appear in a `evl_poll_event` entry in _pollset_. Otherwise, a negated
error code is returned:

- -EBADF	 _efd_ is not a valid file descriptor referring to an EVL
  		 polling set.

- -EINVAL	 _nrset_ is negative.

- -EAGAIN	_fd_ is marked as [non-blocking (O_NONBLOCK)]
  		(http://man7.org/linux/man-pages/man2/fcntl.2.html), and no
  		file descriptor from the polling set has any event pending.

- -EFAULT	 _pollset_ points to invalid memory.

---

{{< proto evl_timedpoll >}}
int evl_timedpoll(int efd, struct evl_poll_event *pollset, int nrset, struct timespec *timeout)
{{< /proto >}}

This call is a variant of [evl_poll()]({{% relref "#evl_poll" %}})
which allows specifying a timeout on the polling operation, so that
the caller is unblocked after a specified delay sleeping without
receiving any event.

{{% argument efd %}}
The file descriptor of a polling set obtained by a previous call to
[evl_new_poll()]({{% relref "#evl_new_poll" %}}).
{{% /argument %}}

{{% argument pollset %}}
The start of an array of structures of type `evl_poll_event`
containing the set of file descriptors which have received at least
one event when the call returns.
{{% /argument %}}

{{% argument nrset %}}
The number of structures in the _pollset_ array. If zero,
[evl_poll()]({{% relref "#evl_new_poll" %}}) returns immediately with
a zero count.
{{% /argument %}}

{{% argument timeout %}}
A time limit to wait for any event to be notified before the call
returns on error. Timeouts are always based on the built-in
[EVL_CLOCK_MONOTONIC]({{< relref
"core/user-api/clock/_index.md#builtin-clocks" >}}) clock. If both the
`tv_sec` and `tv_nsec` members of _timeout_ are set to zero,
[evl_timedpoll()]({{% relref "#evl_timedpoll" %}}) behaves identically
to [evl_poll()]({{% relref "#evl_poll" %}}).
{{% /argument %}}

On success, `evl_timedpoll()` returns the count of file descriptors
from the polling set for which an event is pending, each of them will
appear in a `evl_poll_event` entry in _pollset_. Otherwise, a negated
error code is returned:

- -EBADF	 _efd_ is not a valid file descriptor referring to an EVL
  		 polling set.

- -EINVAL	 _nrset_ is negative.

- -EFAULT	 _pollset_ or _timeout_ point to invalid memory.

- -EINVAL	 any of `tv_sec` or `tv_nsec` in _timeout_ is invalid.

- -ETIMEDOUT	 The timeout fired, after the amount of time specified by
		 _timeout_.

---

{{<lastmodified>}}
