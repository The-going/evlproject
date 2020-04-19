---
title: "API revisions"
weight: 100
---

You can obtain the current API revision of [libevl]({{< relref
"core/user-api/_index.md" >}}) either at compilation time using the
value of the `__EVL__` macro, or dynamically by calling
[evl_get_version()]({{< relref
"core/user-api/misc/_index.md#evl_get_version" >}}).

### rev. 12 ([libevl r15](https://git.evlproject.org/libevl.git/tag/?h=r15))

[Element visibility is introduced]({{< relref
"core/user-api/_index.md#element-visibility" >}}), as a result:

- Each element class provides a new long-form evl\_create\_*() call,
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
  "core/user-api/xbuf/_index.md#evl_create_xbuf" >}}).

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

### rev. 11 ([libevl r14](https://git.evlproject.org/libevl.git/tag/?h=r14))

For naming consistency, evl_sched_control() was renamed
[evl_control_sched()]({{< relref
"core/user-api/scheduling/_index.md#evl_control_sched" >}}).

---

{{<lastmodified>}}
