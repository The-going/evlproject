---
menuTitle: "Thread"
weight: 5
---

### Thread element {#thread-element}

The main kernel's thread is the basic execution unit in EVL. The most
common kind of EVL threads is a regular POSIX thread started by
`pthread_create(3)` which has attached itself to the EVL core by a
call to [`evl_attach_self()`]({{% relref
"core/user-api/thread/routines/_index.md#evl_attach_self" %}}). Once a
POSIX thread attached itself to EVL, it can:

- request real-time services to the core, exclusively by calling
  routines available from the EVL library . In this case, and only in
  this one, you get real-time guarantees for the caller. This is what
  time-critical processing loops are supposed to use. Such request may
  switch the calling thread to the out-of-band [execution
  stage]({{%relref "dovetail/pipeline/_index.md#two-stage-pipeline"
  %}}), for running under EVL's supervision in order to ensure
  real-time behaviour.

{{% notice info %}}
A thread which is being scheduled by EVL instead of the main kernel is
said to be running **out-of-band**, as defined by [Dovetail]({{%
relref "dovetail/pipeline/_index.md" %}}). It remains in this mode
until it asks for a service which the main kernel provides.
{{% /notice %}}

- invoke services from your favourite C library (_glibc_, _musl_,
  _uClibc_ etc.), which may end up issuing system calls to the main
  kernel for carrying out the job. EVL may have to demote the caller
  automatically from the EVL context to a runtime mode which is
  compatible with using the main kernel services. As a result of this,
  you get NO help from EVL for keeping short and bounded latency
  figures anymore, but you do have access to any feature the main
  kernel provides. This mode is normally reserved to initialization
  and cleanup phases of your application. If you end up using them in
  the middle of a would-be time-critical loop, well, something is
  seriously wrong in this code.

{{% notice info %}}
A thread which is being scheduled by the main kernel instead of EVL is
said to be running **in-band**, as defined by [Dovetail]({{% relref
"dovetail/pipeline/_index.md" %}}). It remains in this mode until it
asks for a service which EVL can only provide when the caller runs
out-of-band.
{{% /notice %}}

---

### Thread services

#### int evl_attach_self(const char *_fmt_, ...) {#evl_attach_self}

EVL does not really _create_ threads; instead, it enables a regular
POSIX thread to invoke its real-time services once this thread has
attached to the EVL core.  `evl_attach_self()` is the library call
which requests such attachment. All you need to provide is a _unique_
thread name, which will be the name of the device element representing
that thread in the file system.

There is no requirement as of when `evl_attach_self()` should be
called in a thread's execution flow. You just have to call it before
it starts requesting EVL services.

{{% notice note %}}
The `main()` thread of a process is no different from any other thread
to EVL. It may call `evl_attach_self()` whenever you see fit, or not
at all if you don't plan to request EVL services from this context.
{{% /notice %}}

##### Arguments

_fmt_	A printf-like format string to generate the thread name. A
common way of generating unique thread names is to add the calling
process's _pid_ somewhere into the format string as illustrated in the
example.

{{% notice warning %}}
The generated name is used to form a file path, referring to the new
thread element's device in the file system. So this name must contain
only valid characters in this context, excluding slashes.
{{% /notice %}}

...	The optional variable argument list completing the format.

##### Return value

`evl_attach_self()` returns the file descriptor of the newly attached
thread on success. You may use this _fd_ to submit requests for the
newly attached thread in other calls from the EVL library which ask
for a thread file descriptor.

If the call fails, a negated error code is returned:

-EEXIST		The generated name is conflicting with an existing thread's
		name.

-EINVAL		The generated name is badly formed, likely containing
		invalid character(s), such as a slash. Keep in mind that
		it should be usable as a basename of a device element's file path.

-ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

-EMFILE		The per-process limit on the number of open file descriptors
		has been reached.

-ENFILE		The system-wide limit on the total number of open files
		has been reached.

-EPERM		The caller is not allowed to lock memory via a call to
		`mlockall(2)`. Since memory locking is a requirement for running
		EVL threads, no joy.

-ENOMEM		No memory available, whether the kernel could not
		lock down all of the calling process's virtual address
		space into RAM, or some other reason related to some
		process or driver eating way too much virtual or physical
		memory.	You may start panicking.

-ENOSYS		There is no EVL core enabled in the running kernel.

##### Example

```
#include <evl/thread.h>

static void *byte_crunching_thread(void *arg)
{
	int efd;

	/* Attach the current thread to the EVL core. */
	efd = evl_attach_self("cruncher-%d", getpid());
	...
}
```

As a result of this call, you should see a new device appear into the
_/dev/evl/thread_ hierarchy, e.g.:

```
$ ls -l /dev/evl/thread
total 0
crw-rw----    1 root     root      246,   1 Jan  1  1970 clone
crw-rw----    1 root     root      246,   1 Jan  1  1970 cruncher-2712
```

##### Notes

1. You can revert the attachment to EVL at any time by calling
[`evl_detach_self()`]({{% relref
"core/user-api/thread/routines/_index.md#evl_detach_self" %}}) from the context
of the thread to detach.

2. Closing all the file descriptors referring to an EVL thread is not
enough to drop its attachment to the EVL core. It merely prevents to
submit any further request for the original thread via calls taking
file descriptors. You would still have to call
[`evl_detach_self()`]({{% relref
"core/user-api/thread/routines/_index.md#evl_detach_self" %}}) from
the context of this thread to fully detach it.

3. If a valid file descriptor is still referring to a detached thread,
or after the thread has exited, any request submitted for that thread
using such _fd_ would receive -ESTALE.

4. An EVL thread which exits is automatically detached from the EVL
core, you don't have to call [`evl_detach_self()`]({{% relref
"core/user-api/thread/routines/_index.md#evl_detach_self" %}})
explicitly before exiting your thread.

5. The EVL core drops the kernel resources attached to a thread once
it has detached itself or has exited, and only after all the file
descriptors referring to that thread have been closed.

6. The EVL library sets the O_CLOEXEC flag on the file descriptor
referring to the newly attached thread before returning from
`evl_attach_self()`.

---

#### int evl_detach_self(void) {#evl_detach_self}

`evl_detach_self()` reverts the action of [`evl_attach_self()`]({{%
relref "core/user-api/thread/routines/_index.md#evl_attach_self" %}}),
detaching the calling thread from the EVL core. Once this operation
has succeeded, the current thread cannot submit EVL requests anymore.

##### Return value

This call returns zero on success, or a negated error code if something
went wrong:

-EPERM		The current thread is not attached to the EVL core.

##### Example

```
#include <evl/thread.h>

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

##### Notes {#detached-thread-notes}

1. You can re-attach the detached thread to EVL at any time by calling
[`evl_attach_self()`]({{% relref
"core/user-api/thread/routines/_index.md#evl_attach_self" %}}) again.

2. If a valid file descriptor is still referring to a detached thread,
or after the thread has exited, any request submitted for that thread
using such descriptor would receive -ESTALE.

3. An EVL thread which exits is automatically detached from the EVL
core, you don't have to call `evl_detach_self()` explicitly before
exiting your thread.

4. The EVL core drops the kernel resources attached to a thread once
it has detached itself or has exited, and only after all the file
descriptors referring to that thread have been closed.

---

#### int evl_get_self(void)

`evl_get_self()` returns the file descriptor obtained for the current
thread after a successful call to [`evl_attach_self()`]({{% relref
"core/user-api/thread/routines/_index.md#evl_attach_self" %}}).  You
may use this _fd_ to submit requests for the current thread in other
calls from the EVL library which ask for a thread file descriptor.

##### Return value

This call returns a valid file descriptor referring to the caller on
success, or a negated error code if something went wrong:

-EPERM		The current thread is not attached to the EVL core.

##### Example

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

##### Note

`evl_get_self()` will fail after a call to [`evl_detach_self()`]({{%
relref "core/user-api/thread/routines/_index.md#evl_detach_self" %}}).

---

#### int evl_switch_oob(void) {#evl_switch_oob}

Applications are unlikely to ever use this call explicitly: it
switches the calling thread to the out-of-band [execution
stage]({{%relref "dovetail/pipeline/_index.md#two-stage-pipeline"
%}}), for running under EVL's supervision which ensures real-time
behaviour. Any EVL service which requires it will enforce such switch
if and when required automatically, so in most cases there should be
no point in dealing with this manually in applications.

`evl_switch_oob()` is defined for the rare circumstances where some
high-level API based on the EVL core library might have to enforce a
particular execution stage, based on a deep knowledge of how EVL works
internally. Entering a syscall-free section of code for which the
out-of-band mode needs to be guaranteed on entry would be the only
valid reason to call `evl_switch_oob()`.

{{% notice warning %}}
Forcing the current execution stage between in-band and out-of-band
stages is a heavyweight operation: this entails two thread context
switches both ways, as the switching thread is offloaded to the
opposite scheduler. You really don't want to force this explicitly
unless you definitely have to and fully understand the implications of
it runtime-wise. Bottom line is that **calling a main kernel service
from within a time-critical code is a clear indication that something
is wrong** in such code. This invalidates the reason why a
time-critical code would need to switch back to out-of-band mode
eagerly.
{{% /notice %}}

##### Return value

This call returns zero on success, or a negated error code if
something went wrong:

-EPERM		The current thread is not attached to the EVL core.

---

#### int evl_switch_inband(void)  {#evl_switch_inband}

Applications are unlikely to ever use this call explicitly: it
switches the calling thread to the in-band [execution stage]({{%relref
"dovetail/pipeline/_index.md#two-stage-pipeline" %}}), for running
under the main kernel supervision. Any EVL thread which issues a
system call to the main kernel will be switched to the in-band context
automatically, so in most cases there should be no point in dealing
with this manually in applications.

`evl_switch_inband()` is defined for the rare circumstances where some
high-level API based on the EVL core library might have to enforce a
particular execution stage, based on a deep knowledge of how EVL works
internally. Entering a syscall-free section of code for which the
in-band mode needs to be guaranteed on entry would be the only valid
reason to call `evl_switch_inband()`.

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

##### Return value

This call returns zero on success, or a negated error code if
something went wrong:

-EPERM		The current thread is not attached to the EVL core.

---

#### int evl_set_schedattr(int efd, const struct evl_sched_attrs *attrs) {#evl_set_schedattr}

This call changes the scheduling attributes for the thread referred to
by _efd_ in the EVL core.

_efd_	A file descriptor referring to the target thread, as returned by
	`evl_attach_self()`, `evl_get_self()`, or opening a thread
	element device in _/dev/evl/thread_ using `open(2)`.

_attrs_ A structure defining the new set of attributes, which depends
	on the scheduling policy mentioned in
	`attrs->sched_policy`. EVL currently implement the following
	policies:

- [SCHED_FIFO]({{% relref
  "core/user-api/scheduling/_index.md#SCHED_FIFO" %}}), which is the
  common first-in, first-out real-time policy.

- [SCHED_RR]({{% relref "core/user-api/scheduling/_index.md#SCHED_RR"
  %}}), defining a real-time, round-robin policy in which each member
  of the class is allotted an individual time quantum before the CPU
  is given to the next thread.

- [SCHED_QUOTA]({{% relref
  "core/user-api/scheduling/_index.md#SCHED_QUOTA" %}}), which
  enforces a limitation on the CPU consumption of threads over a fixed
  period of time, known as the global quota period. Threads undergoing
  this policy are pooled in groups, with each group being given a
  share of the period.

- [SCHED_WEAK]({{% relref
  "core/user-api/scheduling/_index.md#SCHED_WEAK" %}}), which is a
  *non real-time* policy allowing its members to run in-band most of
  the time, while retaining the ability to request EVL services, at
  the expense of briefly switching to the out-of-band execution stage
  on demand.

##### Return value

`evl_sched_attrs()` returns zero on success, otherwise a negated error
code is returned:

-EBADF		_efd_ is not a valid thread descriptor.

-EINVAL		Some of the parameters in _attrs_ are wrong. Check
		`attrs->sched_policy`, and the policy-specific
		information may EVL expect for more.

-ESTALE _efd_ refers to a stale thread, see these [notes]({{% relref
 "core/user-api/thread/routines/_index.md#detached-thread-notes" %}}).

##### Example

```
#include <evl/thread.h>

int change_self_schedparams(void)
{
	struct evl_sched_attrs attrs;
	int ret;

	attrs.sched_policy = SCHED_FIFO;
	attrs.sched_priority = 90;
	return evl_set_schedattr(evl_get_self(), &attrs);
}
```

##### Note

`evl_set_schedattr()` immediately changes the scheduling attributes
the EVL core uses for the target thread when it runs in out-of-band
context. Later on, the next time such thread transitions from
out-of-band to in-band context, the main kernel will apply an
extrapolated version of those changes to its own scheduler as well.

{{% notice note %}}
Calling `pthread_setschedparam()` from the C library does not affect
the scheduling attributes of an EVL thread. It only affects the
scheduling parameters of such thread from the standpoint of the main
kernel.
{{% /notice %}}

The extrapolation of the out-of-band scheduling attributes passed to
`evl_set_schedattr()` to the in-band ones applied by the mainline
kernel works as follows:

| out-of-band policy    | in-band policy |
| ------------------    | -------------- |
| SCHED_FIFO, prio      |    SCHED_FIFO, prio  |
| SCHED_RR, prio        |    SCHED_FIFO, prio  |
| SCHED_WEAK, prio > 0  |    SCHED_FIFO  |
| SCHED_WEAK, prio == 0 |    SCHED_OTHER |

{{% notice warning %}}
The C library may cache the current scheduling attributes for the
in-band context of a thread, _glibc_ does so typically. Since EVL
passes on the changes to the main kernel only, the cached value may
not reflect the actual scheduling attributes of the thread anymore.
{{% /notice %}}

---

#### int evl_get_schedattr(int efd, struct evl_sched_attrs *attrs)

---

#### int evl_get_state(int efd, struct evl_thread_state *statebuf)

---

### Can EVL threads run in kernel space?

Yes. Drivers may create kernel-based EVL threads backed by regular
_kthreads_, using EVL's [kernel API]({{% relref
"core/kernel-api/_index.md" %}}). The attachment phase is hidden
inside the API call starting the EVL kthread in this case. Most of the
notions explained in this document apply to them too, except that
there is no system call interface between the EVL core and the
kthread. **So nothing can prevent EVL kthreads from calling the main
kernel services from the wrong context.**

### Where do thread devices live?

Each time a new thread element is created, it appears into the
_/dev/evl/thread_ hierarchy, e.g.:

```
$ ls -l /dev/evl/threads
total 0
crw-rw----    1 root     root      246,   1 Jan  1  1970 clone
crw-rw----    1 root     root      244,   0 Mar  1 11:26 rtk1@0:1682
crw-rw----    1 root     root      244,  18 Mar  1 11:26 rtk1@1:1682
crw-rw----    1 root     root      244,  36 Mar  1 11:26 rtk1@2:1682
crw-rw----    1 root     root      244,  54 Mar  1 11:26 rtk1@3:1682
crw-rw----    1 root     root      244,   1 Mar  1 11:26 rtk2@0:1682
crw-rw----    1 root     root      244,  19 Mar  1 11:26 rtk2@1:1682
crw-rw----    1 root     root      244,  37 Mar  1 11:26 rtk2@2:1682
crw-rw----    1 root     root      244,  55 Mar  1 11:26 rtk2@3:1682
                           (snip)
crw-rw----    1 root     root      244,   9 Mar  1 11:26 rtus_ufps0-10:1682
crw-rw----    1 root     root      244,   8 Mar  1 11:26 rtus_ufps0-9:1682
crw-rw----    1 root     root      244,  27 Mar  1 11:26 rtus_ufps1-10:1682
crw-rw----    1 root     root      244,  26 Mar  1 11:26 rtus_ufps1-9:1682
crw-rw----    1 root     root      244,  45 Mar  1 11:26 rtus_ufps2-10:1682
crw-rw----    1 root     root      244,  44 Mar  1 11:26 rtus_ufps2-9:1682
crw-rw----    1 root     root      244,  63 Mar  1 11:26 rtus_ufps3-10:1682
crw-rw----    1 root     root      244,  62 Mar  1 11:26 rtus_ufps3-9:1682
```

{{% notice info %}}
The _clone_ file is a special device which allows the EVL library to
request the creation of a thread element. _This is for internal use only_.
{{% /notice %}}

### How to reach a remote EVL thread?

If you need to submit requests for an EVL thread which belongs to a
different process, you only need to open the device file representing
this element in _/dev/evl/threads_, then use the file descriptor
just obtained in the thread-related request you want to send it. For
instance, we could change the scheduling parameters of an EVL kernel
thread named rtk1@3:1682 from a companion application in userland as
follows:

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

### Where to look for thread information?

Since every element is backed by a regular character device, the place
to look for thread attributes is in the /sysfs hierarchy, where such
device is living. For instance, we can have a look at the attributes
exported by the sampling thread of EVL's _latmus_ utility like this:

```
cd /sys/devices/virtual/thread/lat-sampler:2136
# ls -l
total 0
-r--r--r--    1 root     root          4096 Mar  1 12:01 pid
-r--r--r--    1 root     root          4096 Mar  1 12:01 sched
-r--r--r--    1 root     root          4096 Mar  1 12:01 state
-r--r--r--    1 root     root          4096 Mar  1 12:01 stats
-r--r--r--    1 root     root          4096 Mar  1 12:01 timeout

# cat pid sched state stats timeout
2140
0 90 90 rt
0x8002
1 311156 311175 46999122352 0
0
```
