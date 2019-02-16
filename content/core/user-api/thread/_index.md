---
title: "Thread element"
menuTitle: "Threads"
weight: 5
---

## What is that?

This is the basic execution unit in EVL, which is the same as the main
kernel's. The most common kind of EVL threads is a regular POSIX
thread started by `pthread_create(3)` which has attached itself to the
EVL core by a call to [`evl_attach_self()`]({{% relref
"core/user-api/thread/evl_attach_self/_index.md" %}}). Once a POSIX
thread attached itself to EVL, it can:

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


## Can EVL threads run in kernel space?

Yes. Drivers may create kernel-based EVL threads backed by regular
_kthreads_, using EVL's [kernel API]({{% relref
"core/kernel-api/_index.md" %}}). The attachment phase is hidden
inside the API call starting the EVL kthread in this case. Most of the
notions explained in this document apply to them too, except that
there is no system call interface between the EVL core and the
kthread. **So nothing can prevent EVL kthreads from calling the main
kernel services from the wrong context.**

## Where do threads live?

Each time a new thread element is created, it appears into the
_/dev/evenless/thread_ hierarchy, e.g.:

```
$ ls -l /dev/evenless/threads
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

## How to talk to a remote EVL thread?

If you need to submit requests for an EVL thread which belongs to a
different process, you only need to open the device file representing
this element in _/dev/evenless/threads_, then use the file descriptor
just obtained in the thread-related request you want to send it. For
instance, we could change the scheduling parameters of an EVL kernel
thread named rtk1@3:1682 from a companion application in userland as
follows:

```
	struct evl_sched_attrs attrs;
	int efd, ret;

 	efd = open("/dev/evenless/thread/rtk1@3:1682", O_RDWR);
	/* skipping checks */

	attrs.sched_policy = SCHED_FIFO;
	attrs.sched_priority = 90;
	ret = evl_set_schedattr(efd, &attrs);
	/* skipping checks */
```

## Where to look for thread information?

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
