---
title: "Retrieve a thread's self handle"
menuTitle: "evl_get_self"
weight: 35
---

### int evl_get_self(void)

`evl_get_self()` returns the file descriptor obtained for the current
thread after a successful call to [`evl_attach_self()`]({{% relref
"core/user-api/thread/evl_attach_self/_index.md" %}}).  You may use
this _fd_ to submit requests for the current thread in other calls
from the EVL library which ask for a thread file descriptor.

### Return value

This call returns a valid file descriptor referring to the caller on
success, or a negated error code if something went wrong:

-EPERM		The current thread is not attached to the EVL core.

Anything else is likely an EVL bug. Congratulations.

### Example

```
#include <evenless/thread.h>

static void get_caller_info(void)
{
	struct evl_thread_state statebuf;
	int efd, ret;

	/* Fetch the current thread's _fd_. */
	efd = evl_get_self();
	...
	/* Retrieve the caller's state information. */
	ret = evl_get_state(efd, &statebuf);
	...
}
```

### Note

`evl_get_self()` will fail after a call to [`evl_detach_self()`]({{%
relref "core/user-api/thread/evl_etach_self/_index.md" %}}).
