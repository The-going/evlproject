---
menuTitle: "Thread"
weight: 5
---

### Thread element {#thread-element}

The main kernel's thread is the basic execution unit in EVL. The most
common kind of EVL threads is a regular POSIX thread started by
[pthread_create(3)](http://man7.org/linux/man-pages/man3/pthread_create.3.html)
which has attached itself to the EVL core by a call to
[evl_attach_self()]({{% relref "#evl_attach_self" %}}). Once a POSIX
thread attached itself to EVL, it can:

- request real-time services to the core, exclusively by calling
  routines available from the EVL library . In this case, and only in
  this one, you get real-time guarantees for the caller. This is what
  time-critical processing loops are supposed to use. Such request may
  switch the calling thread to the out-of-band [execution
  stage]({{%relref "dovetail/pipeline/_index.md#two-stage-pipeline"
  %}}), for running under EVL's supervision in order to ensure
  real-time behaviour.

- invoke services from your favourite C library (_glibc_, _musl_,
  _uClibc_ etc.), which may end up issuing system calls to the main
  kernel for carrying out the job. EVL may have to demote the caller
  automatically from the EVL context to the in-band stage, so that it
  enters a runtime mode which is compatible with using the main kernel
  services. As a result of this, you get NO help from EVL for keeping
  short and bounded latency figures anymore, but you do have access to
  any feature the main kernel provides. This mode is normally reserved
  to initialization and cleanup phases of your application. If you end
  up using them in the middle of a would-be time-critical loop, well,
  something is seriously wrong in this code.

{{% notice info %}}
A thread which is being scheduled by EVL instead of the main kernel is
said to be running _out-of-band_, as defined by [Dovetail]({{%
relref "dovetail/pipeline/_index.md" %}}). It remains in this mode
until it asks for a service which the main kernel provides.
Conversely, a thread which is being scheduled by the main kernel
instead of EVL is said to be running _in-band_, as defined by
[Dovetail]({{% relref "dovetail/pipeline/_index.md" %}}). It remains
in this mode until it asks for a service which EVL can only provide
to the caller when it runs out-of-band.
{{% /notice %}}

### Thread services {#thread-services}

---

{{< proto evl_attach_thread >}}
int evl_attach_thread(int flags, const char *fmt, ...)
{{< /proto >}}

EVL does not actually _create_ threads; instead, it enables a regular
POSIX thread to invoke its real-time services once this thread has
attached to the EVL core.  [evl_attach_thread()]({{% relref
"#evl_attach_thread" %}}) is the initial service which requests such
attachment. In most cases, applications would use the
[evl_attach_self()]({{% relref "#evl_attach_self" %}}) shorthand
instead, which calls [evl_attach_thread()]({{% relref
"#evl_attach_thread" %}}) under the hood with the default set of
creation flags.

There is no requirement as of when [evl_attach_thread()]({{% relref
"#evl_attach_thread" %}}) (or [evl_attach_self()]({{% relref
"#evl_attach_self" %}})) should be called in the thread execution
flow. You just have to call it before it starts requesting other EVL
services. Note that the main thread of a process is no different from
any other thread to EVL. It may call [evl_attach_thread()]({{% relref
"#evl_attach_thread" %}}) whenever you see fit, or not at all if you
don't plan to request EVL services from this context.

As part of the attachment process, the calling thread is also pinned
on its current CPU. You may change this default affinity by calling
[sched_setaffinity(2)](http://man7.org/linux/man-pages/man2/sched_setaffinity.2.html)
as you see fit any time after [evl_attach_thread()]({{% relref
"#evl_attach_thread" %}}) has returned, but keep in mind that such
_libc_ service will trigger a common Linux system call, which will
cause your thread to switch to [in-band context]({{< relref
"dovetail/altsched.md#inband-switch" >}}) automatically when doing
so. So you may want to avoid calling
[sched_setaffinity(2)](http://man7.org/linux/man-pages/man2/sched_setaffinity.2.html)
from your time-critical loop, which would not make much sense anyway
since this is fundamentally an heavyweight operation kernel-wise.

As part of the attachment process, the in-band scheduling settings
your thread had before the call is translated to the closest EVL
counterpart, as follows:

| in-band settings        | out-of-band settings |
| ------------------      | ------------------   |
| SCHED_OTHER, 0          |    [SCHED_WEAK]({{% relref "core/user-api/scheduling/_index.md#SCHED_WEAK" %}}), 0     |
| SCHED_BATCH, 0          |    [SCHED_WEAK]({{% relref "core/user-api/scheduling/_index.md#SCHED_WEAK" %}}), 0     |
| SCHED_IDLE, 0           |    [SCHED_WEAK]({{% relref "core/user-api/scheduling/_index.md#SCHED_WEAK" %}}), 0     |
| \<other policies\>, prio  |    [SCHED_FIFO]({{% relref "core/user-api/scheduling/_index.md#SCHED_FIFO" %}}), prio  |

As a consequence, the thread would still run in-band on return from
[evl_attach_thread()]({{% relref "#evl_attach_thread" %}}) if it was
originally assigned to the `SCHED_OTHER`, `SCHED_BATCH` or
`SCHED_IDLE` classes. Conversely, the thread would run out-of-band on
return from [evl_attach_thread()]({{% relref "#evl_attach_thread" %}}) if
it was originally assigned to any other in-band scheduling class
(e.g. `SCHED_FIFO`).

> 
```
#include <sys/types.h>
#include <unistd.h>
#include <sched.h>
#include <pthread.h>
#include <evl/sched.h>
#include <evl/thread.h>

int main(int argc, char *argv[])
{
	struct sched_param param;
	int ret, tfd;

	param.sched_priority = 8;
	ret = pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
	...

	/* EVL inherits the in-band scheduling params upon attachment. */
	tfd = evl_attach_self("app-main-thread:%d", getpid());

	/*
	 * Now main() is running out-of-band, in the EVL SCHED_FIFO
	 * class at priority 8.
	 */
}

```

{{% argument flags %}}
A set of creation flags for the new element, defining its
[visibility]({{< relref "core/user-api/_index.md#element-visibility"
>}}):

  - `EVL_CLONE_PUBLIC` denotes a public element which is represented
    by a device file in the [/dev/evl]({{< relref
    "core/user-api/_index.md#evl-fs-hierarchy" >}}) file hierarchy, which
    makes it visible to other processes for sharing.
  
  - `EVL_CLONE_PRIVATE` denotes an element which is private to the
    calling process. No device file appears for it in the
    [/dev/evl]({{< relref "core/user-api/_index.md#evl-fs-hierarchy" >}})
    file hierarchy.

  - `EVL_CLONE_OBSERVABLE` denotes a thread which may be observed for
    health monitoring purpose. See
    the [Observable element]({{< relref "core/user-api/observable/_index.md#observable-thread"
    >}}).

  - Only if `EVL_CLONE_OBSERVABLE` is present in _flags_,
    `EVL_CLONE_MASTER` may be added to set the Observable associated
    to the new thread to [master mode]({{< relref
    "core/user-api/observable/_index.md#evl_create_observable" >}}).

  - `EVL_CLONE_NONBLOCK` sets the file descriptor of the new thread in
    non-blocking I/O mode (`O_NONBLOCK`). By default, `O_NONBLOCK` is
    cleared for the file descriptor.
{{% /argument %}}

{{% argument fmt %}}
A [printf](http://man7.org/linux/man-pages/man3/printf.3.html)-like
format string to generate the thread name. See this description of the
[naming convention]
({{< relref "core/user-api/_index.md#element-naming-convention" >}}).
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

[evl_attach_thread()]({{% relref "#evl_attach_thread" %}}) returns the
file descriptor of the newly attached thread on success. You may use
this _fd_ to submit requests for this thread in any call which asks
for a thread file descriptor. If the call fails, a negated error code
is returned instead:

- -EEXIST	The generated name is conflicting with an existing thread
		name.

- -EINVAL	Either _flags_ is wrong, or the [generated name]
  		({{< relref "core/user-api/_index.md#element-naming-convention"
  		>}}) is badly formed.

- -ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

- -EPERM	The caller is not allowed to lock memory via a call to
		[mlockall(2)](http://man7.org/linux/man-pages/man2/mlock.2.html). Since
		memory locking is a requirement for running EVL
		threads, no joy.

- -ENOMEM	No memory available, whether the kernel could not
		lock down all of the calling process's virtual address
		space into RAM, or some other reason related to some
		process or driver eating way too much virtual or physical
		memory.	You may start panicking.

- -ENOSYS	The EVL core is not enabled in the running kernel.

- -ENOEXEC      ABI mismatch error, as reported by
  		[evl_init()]({{< relref "core/user-api/init/_index.md#evl_init" >}}).

```
#include <evl/thread.h>

static void *byte_crunching_thread(void *arg)
{
	int efd;

	/* Attach the current thread to the EVL core. */
	efd = evl_attach_self("/cruncher-%d", getpid());
	...
}
```

As a result of this call, you should see a new device appear into the
[/dev/evl/thread]({{< relref
"core/user-api/_index.md#evl-fs-hierarchy" >}}) hierarchy, e.g.:

```
$ ls -l /dev/evl/thread
total 0
crw-rw----    1 root     root      248,   1 Jan  1  1970 clone
crw-rw----    1 root     root      246,   0 Jan  1  1970 cruncher-2712
```

1. You can revert the attachment to EVL at any time by calling
[evl_detach_self()]({{% relref "#evl_detach_self" %}}) from the
context of the thread to detach.

2. Closing all the file descriptors referring to an EVL thread is not
enough to drop its attachment to the EVL core. It merely prevents to
submit any further request for the original thread via calls taking
file descriptors. You would still have to call
[evl_detach_self()]({{% relref "#evl_detach_self" %}}) from the
context of this thread to fully detach it.

3. If a valid file descriptor is still referring to a detached thread,
or after the thread has exited, any request submitted for that thread
using such _fd_ would receive -ESTALE.

4. An EVL thread which exits is automatically detached from the EVL
core, you don't have to call [evl_detach_self()]({{% relref
"#evl_detach_self" %}}) explicitly before exiting your thread.

5. The EVL core drops the kernel resources attached to a thread once
it has detached itself or has exited, and only after all the file
descriptors referring to that thread have been closed.

6. The EVL library sets the O_CLOEXEC flag on the file descriptor
referring to the newly attached thread before returning from
[evl_attach_thread()]({{% relref "#evl_attach_thread" %}}).

---

{{< proto evl_attach_self >}}
int evl_attach_self(int flags, const char *fmt, ...)
{{< /proto >}}

This call is a shorthand for attaching the calling thread to the EVL
core, with the private [visibility attribute]({{< relref
"core/user-api/_index.md#element-visibility" >}}) set. It is identical
to calling:

```
	evl_attach_thread(EVL_CLONE_PRIVATE, fmt, ...);
```

{{% notice info %}}
Note that if the [generated name] ({{< relref
"core/user-api/_index.md#element-naming-convention" >}}) starts with a
slash ('/') character, `EVL_CLONE_PRIVATE` would be automatically turned
into `EVL_CLONE_PUBLIC` internally.
{{% /notice %}}

---

{{< proto evl_detach_thread >}}
int evl_detach_thread(int flags)
{{< /proto >}}

[evl_detach_thread()]({{% relref "#evl_detach_thread" %}}) reverts the
action of [evl_attach_thread()]({{% relref "#evl_attach_thread" %}}),
detaching the calling thread from the EVL core. Once this operation
has succeeded, the current thread cannot submit EVL requests
anymore. Applications should use the [evl_detach_self()]({{% relref
"#evl_detach_self" %}}) shorthand, which calls
[evl_detach_thread()]({{% relref "#evl_detach_thread" %}}) with
_flags_ set to zero as recommended.

This call returns zero on success, otherwise a negated error code if
something went wrong:

{{% argument flags %}}
This parameter is currently unused and should be passed as zero.
{{% /argument %}}

-EINVAL		_flags_ is not zero.

-EPERM		The current thread is not attached to the EVL core.

```
#include <evl/thread.h>

static void *byte_crunching_thread(void *arg)
{
	int efd;

	/* Attach the current thread to the EVL core (using the long call form). */
	efd = evl_attach_thread(EVL_CLONE_PUBLIC, "cruncher-%d", getpid());
	...
	/* Then detach it (also with the long call form). */
	evl_detach_thread(0);
	...
}
```

1. You can re-attach the detached thread to EVL at any time by calling
[evl_attach_thread()]({{% relref "#evl_attach_thread" %}}) again (or
the [evl_attach_self()]({{% relref "#evl_attach_self" %}}) shorthand).

2. If a valid file descriptor is still referring to a detached thread,
or after the thread has exited, any request submitted for that thread
using such descriptor would receive -ESTALE.

3. An EVL thread which exits is automatically detached from the EVL
core, you don't have to call [evl_detach_thread()]({{% relref
"#evl_detach_thread" %}}) explicitly before exiting your thread.

4. The EVL core drops the kernel resources attached to a thread once
it has detached itself or has exited, and only after all the file
descriptors referring to that thread have been closed.

---

{{< proto evl_detach_self >}}
int evl_detach_self(void)
{{< /proto >}}

This call is a shorthand for detaching the calling thread from the EVL
core. It is identical to calling:

```
	evl_detach_thread(0);
```

---

{{< proto evl_get_self >}}
int evl_get_self(void)
{{< /proto >}}

[evl_get_self()]({{% relref "#evl_get_self" %}}) returns the file
descriptor obtained for the current thread after a successful call to
[evl_attach_thread()]({{% relref "#evl_attach_thread" %}}).  You may use
this _fd_ to submit requests for the current thread in other calls
from the EVL library which ask for a thread file descriptor.  This
call returns a valid file descriptor referring to the caller on
success, otherwise a negated error code if something went wrong:

-EPERM		The current thread is not attached to the EVL core.

```
#include <evl/thread.h>

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

[evl_get_self()]({{% relref "#evl_get_self" %}}) will fail after a
call to [evl_detach_thread()]({{% relref "#evl_detach_thread" %}}).

---

{{< proto evl_switch_oob >}}
int evl_switch_oob(void)
{{< /proto >}}

Applications are unlikely to ever use this call explicitly: it
switches the calling thread to the out-of-band [execution
stage]({{%relref "dovetail/pipeline/_index.md#two-stage-pipeline"
%}}), for running under EVL's supervision which ensures real-time
behaviour. Any EVL service which requires it will enforce such switch
if and when required automatically, so in most cases there should be
no point in dealing with this manually in applications.

[evl_switch_oob()]({{% relref "#evl_switch_oob" %}}) is defined for
the rare circumstances where some high-level API based on the EVL core
library might have to enforce a particular execution stage, based on a
deep knowledge of how EVL works internally. Entering a syscall-free
section of code for which running out-of-band must be guaranteed on
entry would be the only valid reason to call [evl_switch_oob()]({{%
relref "#evl_switch_oob" %}}).  This call returns zero on success,
otherwise a negated error code if something went wrong:

-EPERM		The current thread is not attached to the EVL core.

{{% notice warning %}}
Forcing the current execution stage between in-band and out-of-band
stages is a heavyweight operation: this entails two thread context
switches both ways, as the switching thread is offloaded to the
opposite scheduler. You really don't want to force this explicitly
unless you definitely have to and fully understand the implications of
it runtime-wise. Bottom line is that **calling a main kernel service
from within a time-critical code is a clear indication that something
is wrong** in such code. This invalidates the reason why a
time-critical code would need to switch back to the out-of-band stage
eagerly.
{{% /notice %}}

---

{{< proto evl_switch_inband >}}
int evl_switch_inband(void)
{{< /proto >}}

Applications are unlikely to ever use this call explicitly: it
switches the calling thread to the in-band [execution stage]({{%relref
"dovetail/pipeline/_index.md#two-stage-pipeline" %}}), for running
under the main kernel supervision. Any EVL thread which issues a
system call to the main kernel will be switched to the [in-band
context]({{< relref "dovetail/altsched.md#inband-switch" >}})
automatically, so in most cases there should be no point in dealing
with this manually in applications.

[evl_switch_inband()]({{% relref "#evl_switch_inband" %}}) is defined
for the rare circumstances where some high-level API based on the EVL
core library might have to enforce a particular execution stage, based
on a deep knowledge of how EVL works internally. Entering a
syscall-free section of code for which the in-band mode needs to be
guaranteed on entry would be the only valid reason to call
[evl_switch_inband()]({{% relref "#evl_switch_inband" %}}).  This call
returns zero on success, otherwise a negated error code if something
went wrong:

-EPERM		The current thread is not attached to the EVL core.

{{% notice warning %}}
Forcing the current execution stage between in-band and out-of-band
stages is a heavyweight operation: this entails two thread context
switches both ways, as the switching thread is offloaded to the
opposite scheduler. You really don't want to force this explicitly
unless you definitely have to and fully understand the implications of
it runtime-wise. Bottom line is that **switching the execution stage
to in-band from within a time-critical code is a clear indication that
something is wrong** in such code.
{{% /notice %}}

---

{{< proto evl_is_inband >}}
bool evl_is_inband(void)
{{< /proto >}}

In some cases, you may need to check the current execution stage for
the caller. [evl_is_inband()]({{% relref "#evl_is_inband" %}}) returns
a _true_ boolean value if the caller runs [in-band]({{< relref
"#evl_switch_inband" >}}), _false_ otherwise.

{{% notice note %}}
A POSIX thread which is not currently attached to the EVL core always
receives a _true_ value when issuing this call, which makes sense
since it cannot run out-of-band.
{{% /notice %}}

---

{{< proto evl_get_state >}}
int evl_get_state(int efd, struct evl_thread_state *statebuf)
{{< /proto >}}

[evl_get_state()]({{% relref "#evl_get_state" %}}) is an extended
variant of [evl_get_schedattr()]({{< relref
"core/user-api/scheduling/_index.md#evl_get_schedattr" >}}) for
retrieving runtime information about the state of a thread. The return
buffer is of type `struct evl_thread_state`, which is defined as
follows:

```
struct evl_thread_state {
	struct evl_sched_attrs eattrs;
	int cpu;
};
```

- unlike ({{< relref
"core/user-api/scheduling/_index.md#evl_get_schedattr" >}}), the value
returned in `statebuf->attrs.sched_priority` by [evl_get_state()]({{%
relref "#evl_get_state" %}}) may reflect an ongoing [priority
inheritance/ceiling boost]({{< relref "core/user-api/mutex/_index.md"
>}}).

- `statebuf->cpu` is the CPU the target thread runs on at the time of
  the call.

{{% argument efd %}}
A file descriptor referring to the thread to inquire about.
{{% /argument %}}

{{% argument statbuf %}}
A pointer to the information buffer.
{{% /argument %}}

[evl_get_state()]({{% relref "#evl_get_state" %}}) returns zero on
success, otherwise a negated error code:

-EBADF		_efd_ is not a valid thread descriptor.

-ESTALE		_efd_ refers to a stale thread, see these [notes]({{< relref
		"#evl_detach_thread" >}}).

---

{{< proto evl_unblock_thread >}}
int evl_unblock_thread(int efd)
{{< /proto >}}

Unblocks the thread referred to by _efd_ if it is currently sleeping
on some EVL core [monitor element]({{< relref
"content/core/_index.md#evl-core-elements" >}}), waiting for it to be
signaled/available. In other words, the blocking system call is forced
to fail, and as a result the target thread receives the -EINTR error
on return.

{{% argument efd %}}
A file descriptor referring to the thread to unblock.
{{% /argument %}}

[evl_unblock_thread()]({{% relref "#evl_unblock_thread" %}}) returns
zero on success, otherwise a negated error code:

-EBADF		_efd_ is not a valid thread descriptor.

-ESTALE		_efd_ refers to a stale thread, see these [notes]({{< relref
		"#evl_detach_thread" >}}).

---

{{< proto evl_demote_thread >}}
int evl_demote_thread(int efd)
{{< /proto >}}

Demotes the thread referred to by _efd_ if it is currently running
out-of-band with [real-time scheduling]({{< relref
"core/user-api/scheduling/_index.md" >}}) attributes.

Demoting a thread means to force it out of any real-time scheduling
class, unblock it like [evl_unblock_thread()]({{< relref
"#evl_unblock_thread" >}}) would do, and kick it out of the
out-of-band stage, all in the same move. Once demoted, a thread runs
in-band and undergoes the [SCHED_WEAK]({{< relref
"core/user-api/scheduling/_index.md#SCHED_WEAK" >}}) policy.
[evl_demote_thread()]({{< relref "#evl_unblock_thread" >}}) is a
pretty big hammer you don't want to use lightly; it should be reserved
to specific (read: _desperate_) cases when you have to perform some
aggressive recovery procedure, and/or you want to stop a thread
running out-of-band from hogging a CPU.

{{% argument efd %}}
A file descriptor referring to the thread to demote.
{{% /argument %}}

[evl_demote_thread()]({{% relref "#evl_demote_thread" %}}) returns
zero on success, otherwise a negated error code:

-EBADF		_efd_ is not a valid thread descriptor.

-ESTALE		_efd_ refers to a stale thread, see these [notes]({{< relref
		"#evl_detach_thread" >}}).

---

{{< proto evl_set_thread_mode >}}
int evl_set_thread_mode(int efd, int mask, int *oldmask)
{{< /proto >}}

Each EVL thread has a few so-called _mode bits_ which affect its
behavior depending on whether they are
set. [evl_set_thread_mode()]({{% relref "#evl_set_thread_mode" %}})
can set the following flags when present in _mask_:

- T_WOSS: warn on stage switch
- T_WOLI: warn on locking inconsistency
- T_WOSX: warn on stage exclusion
- T_HMSIG: enable notification of HM events via the SIGDEBUG signal
- T_HMOBS: enable notification of HM events via the built-in observable

See the section about the [health monitoring]({{< relref
"#health-monitoring" >}}) of EVL threads for details about these bits.

If any of T_WOSS, T_WOLI or T_WOSX are present in _mask_ but none of
T_HMSIG or T_HMOBS is, then T_HMSIG is turned on automatically,
enabling notification delivery via the SIGDEBUG signal.

{{% argument efd %}}
A file descriptor referring to the target thread.
{{% /argument %}}

{{% argument mask %}}
A bitmask mentioning the mode bits to set. Zero is valid, and leads
to a no-op. Passing a null _mask_ and a valid _oldmask_ pointer
allows peeking at the mode bits currently set for a thread without
changing them.
{{% /argument %}}

{{% argument oldmask %}}
The address of a bitmask which should collect the previous set of
active mode bits for the thread, before the update. NULL
can be passed to discard this information.
{{% /argument %}}

[evl_set_thread_mode()]({{% relref "#evl_set_thread_mode" %}}) returns
zero on success, otherwise a negated error code:

-EINVAL 	 _mask_ contains invalid mode bits. Setting T_HMOBS for a
		 thread which was not created with the [EVL_CLONE_OBSERVABLE]({{< relref
		 "#evl_create_thread" >}}) attribute set is an error.

-EBADF		_efd_ is not a valid thread descriptor.

-ESTALE		_efd_ refers to a stale thread, see these [notes]({{< relref
		"#evl_detach_thread" >}}).

---

{{< proto evl_clear_thread_mode >}}
int evl_clear_thread_mode(int efd, int mask, int *oldmask)
{{< /proto >}}

[evl_clear_thread_mode()]({{% relref "#evl_clear_thread_mode" %}}) is
the converse call to [evl_set_thread_mode()]({{% relref
"#evl_set_thread_mode" %}}), clearing the mode bits mentioned in
_mask_.

If all of T_WOSS, T_WOLI and T_WOSX are cleared for the thread as a
result, T_HMSIG and T_HMOBS are automatically cleared as well by
[evl_clear_thread_mode()]({{< relref "#evl_clear_thread_mode" >}}).

{{% argument efd %}}
A file descriptor referring to the target thread.
{{% /argument %}}

{{% argument mask %}}
A bitmask mentioning the mode bits to clear. Zero is valid, and leads
to a no-op. Passing a null _mask_ and a valid _oldmask_ pointer
allows peeking at the mode bits currently set for a thread without changing
them.
{{% /argument %}}

{{% argument oldmask %}}
The address of a bitmask which should collect the previous set of
active mode bits for the thread, before the update. NULL
can be passed to discard this information.
{{% /argument %}}

[evl_clear_thread_mode()]({{% relref "#evl_clear_thread_mode" %}}) returns
zero on success, otherwise a negated error code:

-EINVAL		_mask_ contains invalid bits.

-EBADF		_efd_ is not a valid thread descriptor.

-ESTALE		_efd_ refers to a stale thread, see these [notes]({{< relref
		"#evl_detach_thread" >}}).

---

{{< proto evl_subscribe >}}
int evl_subscribe(int ofd, unsigned int backlog_count, int flags)
{{< /proto >}}

This service subscribes the current thread to an [Observable]({{<
relref "core/user-api/observable/_index.md" >}}) element, which makes
the former an observer of the latter. This thread does not have to be
[attached]({{< relref "#evl_attach_thread" >}}) to EVL in order to
subscribe to an Observable. Subscribers are independent from each
other, the target Observable may vanish while subscriptions are still
active, observers can come and go freely. In other words, the
relationship between an Observable and its observers is losely
coupled. However, a thread can only have a single active subscription
to a particular Observable, although it can subscribe to any number of
distinct Observables.

{{% argument ofd %}}
A file descriptor referring to the Observable to subscribe to.
{{% /argument %}}

{{% argument backlog_count %}}
The number of notifications which the core can buffer for this
subscription. On overflow, the unread events already queued are
preserved, the new ones are lost for the observer.
{{% /argument %}}

{{% argument flags %}}
A mask of ORed operation flags which further
qualify the type of subscription. If `EVL_NOTIFY_ONCHANGE` is passed,
the EVL core will merge multiple consecutive notifications for the
same tag and event values. In other words, the returned _( tag, value
)_ pairs will be different at every [receipt]({{< relref
"core/user-api/observable/_index.md#evl_read_observable" >}}). Passing
zero or `EVL_NOTIFY_ALWAYS` ensures that all notices received by the
Observable are passed to this subscriber, unfiltered.
{{% /argument %}}

[evl_subscribe()]({{% relref "#evl_subscribe" %}}) returns
zero on success, otherwise a negated error code:

- -EINVAL 	 _flags_ contains invalid operations bits. The only
		 valid bit is `EVL_NOTIFY_ONCHANGE`, or _backlock\_log\_count_
		 is zero.

- -EBADF	_ofd_ is not a valid file descriptor.

- -EPERM	_ofd_ does not refer to an Observable element.

- -ENOMEM	No memory available for the operation. That is a problem.

---

{{< proto evl_unsubscribe >}}
int evl_unsubscribe(int ofd)
{{< /proto >}}

This service unsubscribes the current thread from an [Observable]({{<
relref "core/user-api/observable/_index.md" >}}) element. This is the
converse call to [evl_subscribe()]({{< relref "#evl_unsubscribe" >}}).

{{% argument ofd %}}
A file descriptor referring to the Observable to unsubscribe from.
{{% /argument %}}

[evl_unsubscribe()]({{% relref "#evl_unsubscribe" %}}) returns
zero on success, otherwise a negated error code:

- -EBADF	_ofd_ is not a valid file descriptor.

- -EPERM	_ofd_ does not refer to an Observable element.

- -ENOENT	the current thread is not subscribed to the Observable
		referred to by _ofd.

---

### Health monitoring of threads {#health-monitoring}

The EVL core has some health monitoring (HM) capabilities, which can
be enabled separately on a per-thread basis using
[evl_set_thread_mode()]({{< relref "#evl_set_thread_mode" >}}), or
global to the system via the kernel configuration. They are based on
runtime error detection when performing user requests which involve
threads. Each type of error is associated with a diagnostic code, such
as:

```
/* Health monitoring diag codes (via observable or SIGDEBUG). */
#define EVL_HMDIAG_SIGDEMOTE	1
#define EVL_HMDIAG_SYSDEMOTE	2
#define EVL_HMDIAG_EXDEMOTE	3
#define EVL_HMDIAG_WATCHDOG	4
#define EVL_HMDIAG_LKDEPEND	5
#define EVL_HMDIAG_LKIMBALANCE	6
#define EVL_HMDIAG_LKSLEEP	7
#define EVL_HMDIAG_STAGEX	8
```

Each of this code identifies a specific cause of trouble for the
thread which receives it:

- `EVL_HMDIAG_SIGDEMOTE`, enabled by the [T_WOSS]({{< relref
  "#evl_set_thread_mode" >}}) mode bit. The thread was demoted to the
  in-band stage because it received a (POSIX) signal. In such an
  event, the core had to release the recipient from any blocked state
  from the out-of-band stage, because handling any pending in-band
  signal is a requirement for the overall system sanity.

- `EVL_HMDIAG_SYSDEMOTE`, enabled by the [T_WOSS]({{< relref
  "#evl_set_thread_mode" >}}) mode bit. The thread was demoted to the
  in-band stage because it issued an in-band Linux syscall, such as
  those defined in your C library of choice. Requesting the in-band
  kernel to handle a system call is by definition a reason to switch
  to in-band execution.

- `EVL_HMDIAG_EXDEMOTE`, enabled by the [T_WOSS]({{< relref
  "#evl_set_thread_mode" >}}) mode bit. The thread was demoted to the
  in-band stage because it received a processor exception while
  running on the out-of-band stage, which it could not handle from
  there. There are different sources of CPU exceptions, the most
  common ones involve invalid memory addressing which typically ends
  up with receiving a SIGSEGV or SIGBUS signal from the kernel as a
  result. When the exception cannot be handled directly from the
  out-of-band stage, the EVL core has to demote the faulting thread so
  that the common (in-band) exception handling code can run safely.

- `EVL_HMDIAG_WATCHDOG`, enabled by [kernel configuration]({{< relref
  "core/build-steps#core-kconfig" >}}). The thread was kicked out of
  out-of-band execution because it hogged a CPU for too long without
  reliquishing it to the in-band stage. The delay applies to the
  entire period while a CPU executes on the out-of-band stage, so this
  may involve multiple EVL threads. Only the thread which is running
  at the time the watchdog expires receives the notification. The
  longer the detection delay (4s by default), the higher the risk of
  breaking the whole system since there is a point when the kernel is
  going to freak out badly if some CPU is unavailable for too long for
  handling in-band work. The timeout delay can be configured using
  [CONFIG_EVL_WATCHDOG]({{< relref "core/build-steps#core-kconfig"
  >}}).

- `EVL_HMDIAG_LKDEPEND`, enabled by the [T_WOLI]({{< relref
  "#evl_set_thread_mode" >}}) mode bit.  There are two converse
  conditions causing this error, both due to an incorrect usage of
  [EVL mutexes]({{< relref "core/user-api/mutex/_index.md" >}})
  leading to a priority inversion:

  1. if a thread is about to switch in-band while owning an EVL mutex
  which is awaited by another thread. This situation would cause a
  priority inversion for the waiter(s), since the latter would depend
  on a mutex owner who lost any guarantee for real-time execution as a
  result of switching in-band.

  2. if a thread running out-of-band is about to sleep on an EVL mutex
  owned by another thread running in-band. Same reasoning as
  previously, an out-of-band thread would depend on the non real-time
  scheduling undergone by the current owner of the mutex.

- `EVL_HMDIAG_LKIMBALANCE`, enabled by the [T_WOLI]({{< relref
  "#evl_set_thread_mode" >}}) mode bit. An attempt to unlock a free
  [EVL mutex]({{< relref "core/user-api/mutex/_index.md" >}}) was
  detected.

- `EVL_HMDIAG_LKSLEEP`, enabled by the [T_WOLI]({{< relref
  "#evl_set_thread_mode" >}}) mode bit. A thread which undergoes the
  [SCHED_WEAK]({{< relref
  "core/user-api/scheduling/_index.md#SCHED_WEAK" >}}) which already
  holds an [EVL mutex]({{< relref "core/user-api/mutex/_index.md" >}})
  subsequently wants to wait for a different type of EVL resource to
  be available, i.e. pretty much any EVL synchronization mechanism
  which may block the caller except EVL mutexes. Such pattern is a bad
  idea: a weakly scheduled thread (EVL-wise, that is) has neither
  real-time requirements nor capabilities, and some real-time thread
  may well wait for it to release the mutex it holds. Therefore,
  waiting for an undefined amount of time for an event to - maybe -
  occur before the mutex can be released eventually is logically
  flawed.

- `EVL_HMDIAG_STAGEX`, enabled by the [T_WOSX]({{< relref
  "#evl_set_thread_mode" >}}) mode bit. A thread is performing an
  out-of-band [I/O operation]({{< relref "core/user-api/io/_index.md"
  >}}) which is blocked on a [stage exclusion lock]({{< relref
  "core/kernel-api/stax/_index.md" >}}) waiting for all in-band tasks
  to leave the section guarded by that lock. This issue leads to a
  priority inversion. [Real-time I/O drivers]({{< relref
  "core/oob-drivers/_index.md" >}}) using stage exclusion should
  provide an interface to applications which enforces a clear
  separation between in-band and out-of-band runtime modes, so that
  this does not normally happen. Blocking on a stax from the
  out-of-band stage might be fine in some circumstances in case
  portions of code are to be shared between in-band and out-of-band
  threads without risking priority inversions, this is the reason why
  such locks exist in the first place. However, this behavior has to
  be specifically allowed by the driver implementation. If
  [T_WOSX]({{< relref "#evl_set_thread_mode" >}}) is set for the
  thread, then such event must be unexpected.

#### `SIGDEBUG` and HM notifications via the observable {#hm-notification-methods}

Once an error condition is detected, the EVL core can notify the
faulting thread by sending it a regular POSIX signal (aka `SIGDEBUG`,
which is `SIGXCPU` in disguise), and/or pushing a notification to the
[observable]({{< relref
"core/user-api/observable/_index.md#observable-thread" >}}) component
of the thread if enabled. Both options are cumulative.

##### Signal-based HM notifications {#hm-sigdebug}

`SIGDEBUG` is enabled by setting the [T_HMSIG]({{< relref
"#evl_set_thread_mode" >}}) mode bit for the thread. A signal handler
should have been installed for receiving them, otherwise the process
would be killed. The macro `sigdebug_cause()` retrieves the diag code
(`EVL_HMDIAG_xxx`) from the SIGDEBUG information block. Checking that
SIGDEBUG was actually sent by the EVL core is recommended, using the
`sigdebug_marked()` macro as illustrated below. If this macro returns
_false_ when passed the signal information block, then your thread has
received `SIGXCPU` from another source, this is _not_ a HM event sent
by the EVL core.

```
#include <signal.h>
#include <evl/thread.h>

/* A basic SIGDEBUG (aka SIGXCPU) handler which only prints out the cause. */

static void sigdebug_handler(int sig, siginfo_t *si, void *context)
{
	if (!sigdebug_marked(si)) {	/* Is this from EVL? */
		you_should_handle_sigxcpu(sig, si, context);
		return;
	}

	/* This is a HM event, handle it. */
	you_should_handle_the_hm_event(sigdebug_cause(si));
}

void install_sigdebug_handler(void)
{
	struct sigaction sa;

	sigemptyset(&sa.sa_mask);
	sa.sa_sigaction = sigdebug_handler;
	sa.sa_flags = SA_SIGINFO;
	sigaction(SIGDEBUG, &sa, NULL);
}
```

`libevl` defines the [evl_sigdebug_handler()]({{< relref
"core/user-api/misc/_index.md#evl_sigdebug_handler" >}}) routine which
simply prints out the diagnostics to _stdout_ then returns.

### Can EVL threads run in kernel space?

Yes. Drivers may create kernel-based EVL threads backed by regular
_kthreads_, using EVL's [kernel API]({{% relref
"core/kernel-api/_index.md" %}}). The attachment phase is hidden
inside the API call starting the EVL kthread in this case. Most of the
notions explained in this document apply to them too, except that
there is no system call interface between the EVL core and the
kthread. For this reason, unlike EVL threads running in user-space,
**nothing prevents EVL kthreads from calling the in-band kernel
routines from the wrong context.**

### Where do public thread devices live?

Each time a new public thread element is created, it appears into the
[/dev/evl/thread]({{< relref
"core/user-api/_index.md#evl-fs-hierarchy" >}}) hierarchy, e.g.:

```
$ ls -l /dev/evl/threads
total 0
crw-rw----    1 root     root      248,   1 Jan  1  1970 clone
crw-rw----    1 root     root      246,   0 Mar  1 11:26 rtk1@0:1682
crw-rw----    1 root     root      246,  18 Mar  1 11:26 rtk1@1:1682
crw-rw----    1 root     root      246,  36 Mar  1 11:26 rtk1@2:1682
crw-rw----    1 root     root      246,  54 Mar  1 11:26 rtk1@3:1682
crw-rw----    1 root     root      246,   1 Mar  1 11:26 rtk2@0:1682
crw-rw----    1 root     root      246,  19 Mar  1 11:26 rtk2@1:1682
crw-rw----    1 root     root      246,  37 Mar  1 11:26 rtk2@2:1682
crw-rw----    1 root     root      246,  55 Mar  1 11:26 rtk2@3:1682
                           (snip)
crw-rw----    1 root     root      246,   9 Mar  1 11:26 rtus_ufps0-10:1682
crw-rw----    1 root     root      246,   8 Mar  1 11:26 rtus_ufps0-9:1682
crw-rw----    1 root     root      246,  27 Mar  1 11:26 rtus_ufps1-10:1682
crw-rw----    1 root     root      246,  26 Mar  1 11:26 rtus_ufps1-9:1682
crw-rw----    1 root     root      246,  45 Mar  1 11:26 rtus_ufps2-10:1682
crw-rw----    1 root     root      246,  44 Mar  1 11:26 rtus_ufps2-9:1682
crw-rw----    1 root     root      246,  63 Mar  1 11:26 rtus_ufps3-10:1682
crw-rw----    1 root     root      246,  62 Mar  1 11:26 rtus_ufps3-9:1682
```

{{% notice info %}}
The _clone_ file is a special device which allows the EVL library to
request the creation of a thread element. _This is for internal use only_.
{{% /notice %}}

### How to reach a remote EVL thread? {#thread-device-open}

If you need to submit requests to an EVL thread which belongs to a
different process, you first need it to have
[public visibility]({{< relref "core/user-api/_index.md#element-visibility" >}}).
If so, then you can open the device file representing
this element in [/dev/evl/thread]({{< relref
"core/user-api/_index.md#evl-fs-hierarchy" >}}), then use the file
descriptor just obtained in the thread-related request you want to
send it. For instance, we could change the scheduling parameters of an
EVL kernel thread named rtk1@3:1682 from a companion application in
userland as follows:

```
	struct evl_sched_attrs attrs;
	int efd, ret;

 	efd = open("/dev/evl/thread/rtk1@3:1682", O_RDWR);
	/* skipping checks */

	attrs.sched_policy = SCHED_FIFO;
	attrs.sched_priority = 90;
	ret = evl_set_schedattr(efd, &attrs);
	/* skipping checks */
```

### Where to look for thread information? {#thread-stats}

#### Using the 'evl ps' command

Running the following command from the shell will report the current
EVL thread activity on your system:

```
# evl ps
CPU   PID   SCHED   PRIO  NAME
  0   398      rt    90   [latmus-klat:394]
  0   399    weak     0   lat-measure:394
```

There are display options you can pass to the ['evl ps']({{< relref
"core/commands.md#evl-ps-command" >}}) command to get more information
about each EVL thread, sorting the result list according to various
criteria.

#### Looking at the /sysfs data {#thread-sysfs-data}

Since every EVL element is backed by a regular character device, so
are threads. Therefore, what to look for is the set of thread device
attributes available from the /sysfs hierarchy. The 'evl ps' command
actually parses this raw information before rendering it in a
human-readable format. Let's have a look at the attributes exported by
the sampling thread of some random run of EVL's _latmus_ utility:

```
# cd /sys/devices/virtual/thread/timer-responder:2136
# ls -l
total 0
-r--r--r--    1 root     root          4096 Mar  1 12:01 pid
-r--r--r--    1 root     root          4096 Mar  1 12:01 sched
-r--r--r--    1 root     root          4096 Mar  1 12:01 state
-r--r--r--    1 root     root          4096 Mar  1 12:01 stats
-r--r--r--    1 root     root          4096 Mar  1 12:01 timeout
-r--r--r--    1 root     root          4096 Mar  1 12:01 observable

# cat pid sched state stats timeout observable
2140
0 90 90 rt
0x8002
1 311156 311175 0 46999122352 0
0
0
```

The format of these fields is as follows:

- _pid_ is the thread identifier (kernel TID); this is a positive
  integer of type pid_t.

- _sched_ contains the scheduling attributes of the thread, with by
  order of appearance:

  - the CPU the thread is running on.
 
  - the current priority level of the thread within its scheduling
    class. With SCHED_FIFO for instance, that would be a figure in the
    [1..99] range. This value may reflect an ongoing priority boost
    due to enforcing the priority inheritance protocol with some EVL
    mutex(es) that thread contends for.

  - the base priority level of the thread within its scheduling class,
    *not* reflecting any priority boost. This is the value that you last
    set with [evl_set_schedattr]({{< relref
    "core/user-api/scheduling/_index.md#evl_set_schedattr" >}}) when
    assigning the thread its scheduling class.

  - the name of the scheduling class the thread is assigned to. This
    is an ASCII string (unquoted), like _rt_ for the SCHED_FIFO class.

  - depending on the scheduling class, you may see optional
    information after the class name which gives some class-specific
    details about the thread. Currently, only SCHED_TP and SCHED_QUOTA
    define such information:

    * SCHED_QUOTA appends the [quota group identifier]({{< relref
    "core/user-api/scheduling/_index.md#create-quota-group" >}}) for
    that thread.

    * SCHED_TP appends the [identifier of the partition]({{< relref
    "core/user-api/scheduling/_index.md#SCHED_TP" >}}) the thread is
    attached to.

- _state_ is the hexadecimal value of the thread's internal state
  word. This information is very ABI dependent, each bit is tersely
  documented in _uapi/evl/thread.h_ from the linux-evl kernel
  tree. This is intended at being food for geek brain.
 
- _stats_ gives statistical information about the CPU consumption of
  the thread, in the following order:

  * the number of (forced) switches to in-band mode, which happens
    when a thread issues an in-band system call from an out-of-band
    context (ISW). This figure should not increase once a real-time
    EVL thread has entered its time-critical work loop, otherwise this
    would mean that such thread is actually leaving the out-of-band
    execution stage while it should not, getting latency hits in the
    process.
   
  * the number of EVL context switches the thread was subject to,
    meaning the number of times the thread was given back the CPU
    after a blocked state (CTXSW). This value _exclusively_ reflects
    the number of switches performed by EVL as a result of resuming
    threads aslept on the out-of-band stage (context switches involved
    in resuming threads aslept on the in-band stage are not _counted_
    here).

  * the number of EVL system calls the thread has issued to the core
    (SYS). Here again, only the EVL system calls are counted, in-band
    system calls from the same threads are tracked by this counter.

  * the number of times the core had to wake up the thread from a
    remote CPU (RWA). This information is useful to find out thread
    placement issues on CPUs.  The best situation is when the core can
    wake up threads directly from the CPU they were put to sleep,
    without inter-processor messaging (IPI) in order to force a remote
    CPU to re-schedule. Although this is not always possible, as
    multiple threads may have to synchronize from distinct CPUs, the
    lesser this number, the smaller the overhead caused by wake up
    requests.

  * the cumulated CPU usage of the thread since its creation,
    expressed as a count of nanoseconds.

  * the percentage of the CPU bandwidth consumed by the thread, from
    the last time this counter was read until the current readout.

- _timeout_ is a count of nanoseconds representing the ongoing delay
  until the thread wakes up from its current timed wait. Zero means no
  timeout. EVL starts a timer when a thread enters a timed wait on
  some kernel resource; _timeout_ reports the time to go until this
  timer fires.

- _observable_ is a boolean value denoting the [observability]({{<
  relref "core/user-api/observable/_index.md#observable-thread" >}})
  of the thread. Non-zero indicates that [EVL_CLONE_OBSERVABLE]({{<
  relref "#evl_create_thread" >}}) was set for this thread, typically
  for [health monitoring]({{< relref "#health_monitoring" >}})
  purpose, which made it observable to itself or to other threads.

---

{{<lastmodified>}}
