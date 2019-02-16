---
title: "Detach a thread"
menuTitle: "evl_detach_self"
weight: 30
---

### int evl_detach_self(void)

`evl_detach_self()` reverts the action of [`evl_attach_self()`]({{%
relref "core/user-api/thread/evl_attach_self/_index.md" %}}),
detaching the calling thread from the EVL core. Once this operation
has succeeded, the current thread cannot submit EVL requests anymore.

### Return value

This call returns zero on success, or a negated error code if something
went wrong:

-EPERM		The current thread is not attached to the EVL core.

Anything else is likely an EVL bug. Congratulations.

### Example

```
#include <evenless/thread.h>

static void *byte_crunching_thread(void *arg)
{
	int efd;

	/* Attach the current thread to the EVL core. */
	efd = evl_attach_self("cruncher-%d", getpid());
	...
	/* Then detach it. */
	evl_detach_self();
	...
}
```

### Notes

1. You can re-attach the detached thread to EVL at any time by calling
[`evl_attach_self()`]({{% relref
"core/user-api/thread/evl_attach_self/_index.md" %}}) again.

2. If a valid file descriptor is still referring to a detached thread
, or after the thread has exited, any request submitted for that
thread using such _fd_ would receive -ESTALE.

3. An EVL thread which exits is automatically detached from the EVL
core, you don't have to call `evl_detach_self()` explicitly before
exiting your thread.

4. The EVL core drops the kernel resources attached to a thread once
it has detached itself or has exited, and only after all the file
descriptors referring to that thread have been closed.
