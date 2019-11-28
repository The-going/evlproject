---
menuTitle: "Initialization"
weight: 1
---

### Initializing `libevl` {#init-libevl}

When initializing, the core EVL library undertakes the following steps
once for any given application process:

- it sets up the specific bits for the CPU architecture, which boils
  down to resolving the address of the
  [vDSO](http://man7.org/linux/man-pages/man7/vdso.7.html) for the
  current process.

- it locks the current and future process memory by a call to
  [mlockall(2)](http://man7.org/linux/man-pages/man2/mlock.2.html), so
  that we won't get page faults due to on-demand memory schemes such
  as the [copy-on-write
  technique](https://landley.net/writing/memory-faq.txt).

- it connects to the EVL core running in kernel space, performing a
  few sanity checks such as making sure the core can handle the ABI
  the application will use later on to issue EVL system calls.

- it initializes a couple of built-in [proxy streams]({{< relref
  "core/user-api/proxy/_index.md" >}}), such as the one
  [evl_printf()]({{< relref "core/user-api/proxy/_index.md#evl_printf"
  >}}) needs to send zero-latency output to _stdout_ from the
  out-of-band context.

- finally, a signal handler is installed in order to receive SIGEVL
  which the core may send to an application process in some (seldom)
  circumstances.

All these actions are performed once by a bootstrap routine named
[evl_init()]({{< relref "#evl_init" >}}), which may be either called
explicitly by the application code - usually at startup on behalf of
the `main()` thread - or indirectly as a result of invoking
[evl_attach_self()]({{% relref
"core/user-api/thread/_index.md#evl_attach_self" %}}) for the first
time. Threads put aside, any attempt to create other EVL objects
before [evl_init()]({{< relref "#evl_init" >}}) has run leads to the
`-ENXIO` error.

> Explicit initialization from the `main()` entry point

```
	#include <error.h>
	#include <evl/evl.h>

	int main(int argc, char *const argv[])
	{
		int ret;

		ret = evl_init();
		if (ret)
			error(1, -ret, "evl_init() failed");
		...
		return 0;
	}
```

> Indirect initialization via [evl_attach_self()]({{% relref
  "core/user-api/thread/_index.md#evl_attach_self" %}})

```
	#include <sys/types.h>
	#include <unistd.h>
	#include <error.h>
  	#include <evl/thread.h>

	int main(int argc, char *const argv[])
	{
		int efd;

		
		efd = evl_attach_self("me-with-pid:%d", getpid());
		if (efd < 0)
			error(1, -efd, "evl_attach_self() failed");
		...
		return 0;
	}
```

### Startup-time services {#startup-services}

---

{{< proto evl_init >}}
int evl_init(void)
{{< /proto >}}

This function boostraps the EVL services for the current process as
explained in the [introduction]({{< relref "#init-libevl" >}}). Only
the first call to this routine applies, subsequent calls to
[evl_init()]({{< relref "#evl_init" >}}) are silently ignored,
returning the status code of the initial call.

On success, this service returns zero. Otherwise, a negated error code
is returned:

- -EPERM	The caller is not allowed to lock memory via a call to
		`mlockall(2)`. Since memory locking is a requirement for running
		EVL threads, no joy.

- -ENOMEM	No memory available, whether the kernel could not
		lock down all of the calling process's virtual address
		space into RAM, or some other reason related to some
		process or driver eating way too much virtual or physical
		memory.	You may start panicking.

- -ENOSYS	The EVL core is not enabled in the running kernel.

- -ENOEXEC      The EVL core in the running kernel exports a different ABI
  		level than the ABI `libevl.so` was compiled
  		against. In some not so distant future, the EVL ABI
  		will be stable enough to provide backward
  		compatibility in the core to applications using older
  		ABI revisions, but we are not there yet. To fix this
  		error in the meantime, you have to rebuild _libevl.so_
  		against the [UAPI the EVL core exports]({{< relref
  		"core/build-steps.md#building-evl-prereq" >}}). If in
  		doubt, rebuild both components, (re-)install the newly
  		built _libevl.so_ and boot on the rebuilt kernel.
