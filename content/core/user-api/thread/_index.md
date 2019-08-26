---
menuTitle: "Thread"
weight: 5
---

### Thread element {#thread-element}

The main kernel's thread is the basic execution unit in EVL. The most
common kind of EVL threads is a regular POSIX thread started by
`pthread_create(3)` which has attached itself to the EVL core by a
call to [`evl_attach_self()`]({{% relref
"core/user-api/thread/_index.md#evl_attach_self" %}}). Once a
POSIX thread attached itself to EVL, it can:

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
  automatically from the EVL context to a runtime mode which is
  compatible with using the main kernel services. As a result of this,
  you get NO help from EVL for keeping short and bounded latency
  figures anymore, but you do have access to any feature the main
  kernel provides. This mode is normally reserved to initialization
  and cleanup phases of your application. If you end up using them in
  the middle of a would-be time-critical loop, well, something is
  seriously wrong in this code.

{{% notice info %}}
A thread which is being scheduled by EVL instead of the main kernel is
said to be running **out-of-band**, as defined by [Dovetail]({{%
relref "dovetail/pipeline/_index.md" %}}). It remains in this mode
until it asks for a service which the main kernel provides.
Conversely, a thread which is being scheduled by the main kernel
instead of EVL is said to be running **in-band**, as defined by
[Dovetail]({{% relref "dovetail/pipeline/_index.md" %}}). It remains
in this mode until it asks for a service which EVL can only provide
to the caller when it runs out-of-band.
{{% /notice %}}

### Thread services {#thread-services}

---

{{< proto evl_attach_self >}}
int evl_attach_self(const char *fmt, ...)
{{< /proto >}}

EVL does not actually _create_ threads; instead, it enables a regular
POSIX thread to invoke its real-time services once this thread has
attached to the EVL core.  evl_attach_self() is the library call which
requests such attachment. All you need to provide is a _unique_ thread
name, which will be the name of the device element representing that
thread in the file system.

There is no requirement as of when evl_attach_self() should be called
in a thread's execution flow. You just have to call it before it
starts requesting EVL services. Note that the main thread of a process
is no different from any other thread to EVL. It may call
evl_attach_self() whenever you see fit, or not at all if you don't
plan to request EVL services from this context.

As part of the attachment process, the calling thread is also pinned
on its current CPU. You may change this default affinity by calling
`sched_setaffinity(2)` as you see fit any time after
`evl_attach_self()` has returned, but keep in mind that such _libc_
service will trigger a regular Linux system call, which will cause
your thread to switch to [in-band context]({{< relref
"dovetail/altsched.md#inband-switch" >}}) automatically when
doing so. So you may want to avoid calling `sched_setaffinity(2)` from
your time-critical loop, which would not make much sense anyway.

{{% argument fmt %}}
A printf-like format string to generate the thread name. A common way
of generating unique names is to add the calling process's
_pid_ somewhere into the format string as illustrated in the
example. The generated name is used to form a file path, referring to
the new thread element's device in the file system. So this name must
contain only valid characters in this context, excluding slashes.
{{% /argument %}}

{{% argument "..." %}}
The optional variable argument list completing the format.
{{% /argument %}}

`evl_attach_self()` returns the file descriptor of the newly attached
thread on success. You may use this _fd_ to submit requests for this
thread in any call which asks for a thread file descriptor. If the
call fails, a negated error code is returned instead:

- -EEXIST	The generated name is conflicting with an existing thread
		name.

- -EINVAL	The generated name is badly formed, likely containing
		invalid character(s), such as a slash. Keep in mind that
		it should be usable as a basename of a device element's file path.

- -ENAMETOOLONG	The overall length of the device element's file path including
		the generated name exceeds PATH_MAX.

- -EMFILE	The per-process limit on the number of open file descriptors
		has been reached.

- -ENFILE	The system-wide limit on the total number of open files
		has been reached.

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

1. You can revert the attachment to EVL at any time by calling
[`evl_detach_self()`]({{% relref
"core/user-api/thread/_index.md#evl_detach_self" %}}) from the context
of the thread to detach.

2. Closing all the file descriptors referring to an EVL thread is not
enough to drop its attachment to the EVL core. It merely prevents to
submit any further request for the original thread via calls taking
file descriptors. You would still have to call
[`evl_detach_self()`]({{% relref
"core/user-api/thread/_index.md#evl_detach_self" %}}) from
the context of this thread to fully detach it.

3. If a valid file descriptor is still referring to a detached thread,
or after the thread has exited, any request submitted for that thread
using such _fd_ would receive -ESTALE.

4. An EVL thread which exits is automatically detached from the EVL
core, you don't have to call [`evl_detach_self()`]({{% relref
"core/user-api/thread/_index.md#evl_detach_self" %}})
explicitly before exiting your thread.

5. The EVL core drops the kernel resources attached to a thread once
it has detached itself or has exited, and only after all the file
descriptors referring to that thread have been closed.

6. The EVL library sets the O_CLOEXEC flag on the file descriptor
referring to the newly attached thread before returning from
`evl_attach_self()`.

---

{{< proto evl_detach_self >}}
int evl_detach_self(void)
{{< /proto >}}

`evl_detach_self()` reverts the action of [`evl_attach_self()`]({{%
relref "core/user-api/thread/_index.md#evl_attach_self" %}}),
detaching the calling thread from the EVL core. Once this operation
has succeeded, the current thread cannot submit EVL requests
anymore. This call returns zero on success, or a negated error code if
something went wrong:

-EPERM		The current thread is not attached to the EVL core.

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

1. You can re-attach the detached thread to EVL at any time by calling
[`evl_attach_self()`]({{% relref
"core/user-api/thread/_index.md#evl_attach_self" %}}) again.

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

{{< proto evl_get_self >}}
int evl_get_self(void)
{{< /proto >}}

`evl_get_self()` returns the file descriptor obtained for the current
thread after a successful call to [`evl_attach_self()`]({{% relref
"core/user-api/thread/_index.md#evl_attach_self" %}}).  You may use
this _fd_ to submit requests for the current thread in other calls
from the EVL library which ask for a thread file descriptor.  This
call returns a valid file descriptor referring to the caller on
success, or a negated error code if something went wrong:

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

`evl_get_self()` will fail after a call to [`evl_detach_self()`]({{%
relref "core/user-api/thread/_index.md#evl_detach_self" %}}).

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

`evl_switch_oob()` is defined for the rare circumstances where some
high-level API based on the EVL core library might have to enforce a
particular execution stage, based on a deep knowledge of how EVL works
internally. Entering a syscall-free section of code for which the
out-of-band mode needs to be guaranteed on entry would be the only
valid reason to call `evl_switch_oob()`.  This call returns zero on
success, or a negated error code if something went wrong:

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
time-critical code would need to switch back to out-of-band mode
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

`evl_switch_inband()` is defined for the rare circumstances where some
high-level API based on the EVL core library might have to enforce a
particular execution stage, based on a deep knowledge of how EVL works
internally. Entering a syscall-free section of code for which the
in-band mode needs to be guaranteed on entry would be the only valid
reason to call `evl_switch_inband()`.  This call returns zero on
success, or a negated error code if something went wrong:

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
the caller. `evl_is_inband()` returns a _true_ boolean value if the
caller runs [in-band]({{< relref "#evl_switch_inband" >}}), _false_
otherwise.

{{% notice note %}}
A POSIX thread which is not currently attached to the EVL core always
receives a _true_ value when issuing this call, which makes sense
since it cannot run out-of-band.
{{% /notice %}}

---

{{< proto evl_get_state >}}
int evl_get_state(int efd, struct evl_thread_state *statebuf)
{{< /proto >}}

`evl_get_state()` is an extended variant of [evl_get_schedattr()]({{<
relref "core/user-api/scheduling/_index.md#evl_get_schedattr" >}}) for
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
returned in `statebuf->attrs.sched_priority` by `evl_get_state()` may
reflect an ongoing [priority inheritance/ceiling boost]({{< relref
"core/user-api/mutex/_index.md" >}}).

- `statebuf->cpu` is the CPU the target thread runs on at the time of
  the call.

`evl_get_state()` returns zero on success, otherwise a negated
error code is returned:

-EBADF		_efd_ is not a valid thread descriptor.

-ESTALE		_efd_ refers to a stale thread, see these [notes]({{< relref
		"#evl_detach_self" >}}).

---

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

#### Using the 'evl ps' command

Running the following command from the shell will report the current
EVL thread activity on your system:

```
# evl ps
CPU   PID   SCHED   PRIO  NAME
  0   398      rt    90   [latmus-klat:394]
  0   399    weak     0   lat-measure:394
```

There are display options you can pass to 'evl ps' to get more
information about each EVL thread, sorting the result list according
to various criteria.

#### Looking at the /sysfs data

Since every EVL element is backed by a regular character device, so
are threads. Therefore, what to look for is the set of thread device
attributes available from the /sysfs hierarchy. The 'evl ps' command
actually parses this raw information before rendering it in a
human-readable format. Let's have a look at the attributes exported by
the sampling thread of some random run of EVL's _latmus_ utility:

```
# cd /sys/devices/virtual/thread/lat-sampler:2136
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
1 311156 311175 0 46999122352 0
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
    the number of switches performed by EVL for resuming the thread in
    out-of-band mode, which excludes any context switch of the same
    thread due to in-band activity.

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
