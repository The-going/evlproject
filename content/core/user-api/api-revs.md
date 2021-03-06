---
title: "API revisions"
weight: 100
---

You can obtain the current API revision of [libevl]({{< relref
"core/user-api/_index.md" >}}) either at compilation time using the
value of the `__EVL__` macro defined in the `<evl/evl.h>` main header
file, or dynamically by calling [evl_get_version()]({{< relref
"core/user-api/misc/_index.md#evl_get_version" >}}).

### rev. 18 ([libevl r26](https://git.xenomai.org/xenomai4/libevl/-/tags/r26))

Introduces the socket interface:

- oob_recvmsg() to receive a message in out-of-band mode.

- oob_sendmsg() to send a message in out-of-band mode.

The regular socket(2) call as extended by ABI 26 is capable of
creating oob-capable sockets when receiving the SOCK_OOB type flag, so
there is no EVL-specific call for this operation.

### rev. 17 ([libevl r17](https://git.xenomai.org/xenomai4/libevl/-/tags/r17))

Enables HM support for threads. Since [ABI 23]({{< relref
"core/abi-revs.md" >}}), the core is able to channel T_WOSS, T_WOLI
and T_WOSX error notifications (SIGDEBUG_xxx) through the [thread
observable]({{< relref
"core/user-api/thread/_index.md#evl_attach_thread" >}}) component if
present. Introduce the T_HMSIG and T_HMOBS mode bits for configuring
the HM notification source(s) of a thread with
[evl_set_thread_mode()]({{< relref
"core/user-api/thread/_index.md#evl_set_thread_mode" >}}).
    
SIGDEBUG_xxx codes are renamed to [EVL_HMDIAG_xxx diag codes]({{<
relref "core/user-api/thread/_index.md#health-monitoring" >}}), so
that we have a single nomenclature for these errors regardless of
whether threads are notified via SIGDEBUG or their observable
component.

### rev. 16 ([libevl r17](https://git.xenomai.org/xenomai4/libevl/-/tags/r17))

Introduces the API changes for supporting the new Observable element:

- adds [evl_subscribe()]({{< relref
  "core/user-api/thread/_index.md#evl_subscribe" >}}) and
  [evl_unsubscribe()]({{< relref
  "core/user-api/thread/_index.md#evl_unsubscribe" >}}) to the thread
  API.

- adds the [evl_create_observable()]({{< relref
  "core/user-api/observable/_index.md#evl_create_observable" >}}),
  [evl_update_observable()]({{< relref
  "core/user-api/observable/_index.md#evl_update_observable" >}}) and
  [evl_read_observable()]({{< relref
  "core/user-api/observable/_index.md#evl_read_observable" >}}) services
  for the new Observable API.

- allows to pass an opaque data to [evl_add_pollfd()]({{< relref
  "core/user-api/poll/_index.md#evl_add_pollfd" >}}) and
  [evl_mode_pollfd()]({{< relref
  "core/user-api/poll/_index.md#evl_mode_pollfd" >}}), which is
  returned into the `struct evl_poll_event` descriptor.

### rev. 15 ([libevl r16](https://git.xenomai.org/xenomai4/libevl/-/tags/r16))

Adds [evl_set_thread_mode()]({{< relref
"core/user-api/thread/_index.md#evl_set_thread_mode" >}}) and
[evl_clear_thread_mode()]({{< relref
"core/user-api/thread/_index.md#evl_set_clear_mode" >}}).

### rev. 14

Adds [evl_unblock_thread()]({{< relref
"core/user-api/thread/_index.md#evl_unblock_thread" >}}) and
[evl_demote_thread()]({{< relref
"core/user-api/thread/_index.md#evl_demote_thread" >}}).

### rev. 13

Adds [evl_yield()]({{< relref
"core/user-api/scheduling/_index.md#evl_yield" >}}).

### rev. 12 ([libevl r15](https://git.xenomai.org/xenomai4/libevl/-/tags/r15))

[Element visibility is introduced]({{< relref
"core/user-api/_index.md#element-visibility" >}}), as a result:

- Most element classes provides a new long-form evl\_create\_*() call,
  in order to receive creation flags. Currently, the visibility
  attribute of elements is the only flag supported (see
  `EVL_CLONE_PRIVATE`, `EVL_CLONE_PUBLIC`). The additional creation
  calls are [evl_create_event()]({{< relref
  "core/user-api/event/_index.md#evl_create_event" >}}),
  [evl_create_flags()]({{< relref
  "core/user-api/flags/_index.md#evl_create_flags" >}}),
  [evl_create_mutex()]({{< relref
  "core/user-api/mutex/_index.md#evl_create_mutex" >}}),
  [evl_create_proxy()]({{< relref
  "core/user-api/proxy/_index.md#evl_create_proxy" >}}),
  [evl_create_sem()]({{< relref
  "core/user-api/sem/_index.md#evl_create_sem" >}}) and
  [evl_create_xbuf()]({{< relref
  "core/user-api/xbuf/_index.md#evl_create_xbuf" >}}).  Likewise, the
  new [evl_attach_thread()]({{< relref
  "core/user-api/thread/_index.md#evl_attach_thread" >}}) and
  [evl_detach_thread()]({{< relref
  "core/user-api/thread/_index.md#evl_detach_thread" >}}) calls
  receive attachment and detachment flags for
  threads. [evl_attach_self()]({{< relref
  "core/user-api/thread/_index.md#evl_attach_self" >}}) is now
  equivalent to attaching a private thread by default, unless the
  thread name [says otherwise]({{< relref
  "core/user-api/_index.md#element-naming-convention"
  >}}). [evl_detach_self()]({{< relref
  "core/user-api/thread/_index.md#evl_detach_self" >}}) is unchanged.

- All evl\_new\_\*() calls become shorthands to their respective
  evl\_create\_*() counterparts, picking reasonable default creation
  parameters for the new element, including private visibility (unless
  overriden by the _leading slash rule_ explained in this
  [document]({{< relref
  "core/user-api/_index.md#element-naming-convention" >}})).

- All long-form evl\_new\_\*_any() calls have been removed from the
  API. Applications should use the corresponding evl\_create\_*() call
  instead.

- [evl_new_proxy()]({{< relref
  "core/user-api/proxy/_index.md#evl_new_proxy" >}}) creates a proxy
  element with no write granularity by default, which caused this this
  parameter to be dropped from the call signature. Applications should
  use [evl_create_proxy()]({{< relref
  "core/user-api/proxy/_index.md#evl_create_proxy" >}}) to specify a
  non-default granularity.

- [evl_new_xbuf()]({{< relref
  "core/user-api/xbuf/_index.md#evl_new_xbuf" >}}) creates a
  cross-buffer element with identically sized input and output buffers
  by default, which caused one of the two size parameters to be
  dropped from the call signature. Applications should use
  [evl_create_xbuf()]({{< relref
  "core/user-api/xbuf/_index.md#evl_create_xbuf" >}}) to specify
  distinct sizes for input and output.

- All former long-form static initializers EVL\_\*\_ANY_INITIALIZER()
  have been renamed EVL\_\*\_INITIALIZER(), dropping the former
  short-form (if any). For instance, EVL_MUTEX_ANY_INITIALIZER() has
  been renamed [EVL_MUTEX_INITIALIZER()]({{< relref
  "core/user-api/mutex/_index.md#EVL_MUTEX_INITIALIZER" >}}), with an
  additional parameter for mentioning both the lock type and
  visibility attribute.

- Selecting the lock type of a mutex is now done using the
  [evl_create_mutex()]({{< relref
  "core/user-api/mutex/_index.md#evl_create_mutex" >}}) call, ORing
  either `EVL_MUTEX_NORMAL` or `EVL_MUTEX_RECURSIVE` into the creation
  flags. This method replaces the _type_ argument to the former
  evl\_new\_mutex\_any() call.

---

### rev. 11 ([libevl r14](https://git.xenomai.org/xenomai4/libevl/-/tags/r14))

For naming consistency, evl_sched_control() was renamed
[evl_control_sched()]({{< relref
"core/user-api/scheduling/_index.md#evl_control_sched" >}}).

---

{{<lastmodified>}}
