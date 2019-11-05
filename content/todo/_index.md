---
menuTitle: "TODO"
title: "TODO list"
weight: 15
pre: "&#9656; "
---

|     Feature summary         |       Status                       |
| :-------------------------- |:---------------------------------- |
|  Export additional core information via [sysfs]({{< relref "#todo-sysfs" >}}) | ![sysfs](/images/unchecked.png?height=30px&width=30px) |
|  Implement [evl ps command]({{< relref "#todo-ps" >}}) | ![ps](/images/checked.png?height=30px&width=30px) |
|  Fix [poll element]({{< relref "#todo-poll-nesting" >}}) for deep nesting | ![poll](/images/unchecked.png?height=30px&width=30px) |
|  Write [unit tests]({{< relref "#todo-libevl-tests" >}}) for libevl | ![tests](/images/unchecked.png?height=30px&width=30px) |
|  Improve [tracepoints]({{< relref "#todo-tracepoints" >}}) in EVL core | ![tracepoints](/images/unchecked.png?height=30px&width=30px) |
|  Kill the [ugly big lock]({{< relref "#todo-kill-biglock" >}}) in EVL core| ![biglock](/images/checked.png?height=30px&width=30px) |
|  Sanity check for [vDSO-based clock source]({{< relref "#todo-vdso-cs" >}}) | ![vdso-cs](/images/unchecked.png?height=30px&width=30px) |

---

### EVL's /sysfs interface needs work {#todo-sysfs}

A choice was made for the EVL core to rely exclusively on /sysfs for
exporting the core state information, excluding /procfs
entirely. After all, we have only a handful of [element types]({{%
relref "core/_index.md#evl-core-elements" %}}) there, each instance is
backed by a regular kernel [device]({{% relref
"core/_index.md#everything-is-a-file" %}}), so using /sysfs attributes
to export information about them to user-space is just the obvious
thing to do.

This said, some work remains to review the attributes already defined
by the EVL core, improve and possibly add more of them. This work is
most likely to be paired with the following task about writing `evl`
status commands.

Missing attributes which come to mind:

- `thread/<pid>/wchan`, which would dump the stack backtrace of the
  corresponding thread if it currently sleeps on an EVL wait channel
  (`struct evl_wait_channel`).

Also, the _sysfs_ element nodes EVL creates appear directly at the top
of the /sys/class hierarchy (e.g. `/sys/class/thread`), which is
clumsy. _sysfs_ does not formally support sub-classes, so I see no
obvious way for having all element classes rooted at `/sys/class/evl`
instead. Any idea would be welcome.

### What the heck is the core running? {#todo-ps}

TASK CLOSED.

As a matter of fact, we have absolutely no handy tool for inspecting
the current runtime status of the EVL core. A _ps_-like command is
direly needed for instance, that would read the /sysfs attributes
exported by the currently running threads to send out some useful
information.

{{% notice tip %}}
The code for an umbrella command named `evl` is already there in
`libevl/utils/evl.c`, which like its `git` counterpart, serves as a
centralized command router for executing ancillary scripts or
binaries. Such a script available from the same directory called
`evl-trace` illustrates the interface logic with the `evl` command.
{{% /notice %}}

### _poll_ is naive with deep nesting {#todo-poll-nesting}

The file descriptor polling feature is represented by a file
descriptor in userland too, so a polling object may be used to poll
other polling objects and so on. The core should make sure not to
accept too deeply nested monitoring requests. The case of cyclic
graphs is detected ok, but non-cyclic deep nesting is definitely not
as the algorithm there is pretty naive. `linux-evl/kernel/evl/poll.c`
is where to look for this.

### libevl needs more tests {#todo-libevl-tests}

There are some unit tests exercising the EVL core in the library, but
we need many more:

- error injection is mostly absent from those tests.

- there is no test verifying the sanity of the condition variable
  implementation. [DONE]

- the file descriptor polling feature is only lightly tested, so is
  the cross-buffer element.  [DONE]

### THE big issue: phase out the ugly big lock {#todo-kill-biglock}

TASK CLOSED.

This lock was inherited from the Xenomai code base (aka _nklock_
there). This is a massive bottleneck on the road to efficient SMP
scalability with more than 6 CPU cores running real-time load.

Some significant preparatory work has taken place in the EVL core for
taking this lock out:

- we don't synchronize the timer events on it anymore, every CPU now
  has its own time base lock, which improves serialization.

- the scheduler core has been simplified significantly, reducing the
  scope of the code which still requires to be covered by the ugly big
  lock.

However, we are not there yet. Some complex stuff remains to be sorted
out for eliminating this lock entirely, specifically in the
thread-to-scheduler interface points.

### Improving the tracepoints {#todo-tracepoints}

The EVL core defines a number of FTRACE-based tracepoints, which for
the most part have been inherited from Xenomai's Cobalt core, there is
root for improvement in order to better fit the EVL context. In
addition, new code which has not been inherited from Xenomai is not
instrumented for the most part.

### Sanity check for OOB-accessible clock_gettime() via vDSO {#todo-vdso-cs}

The latmus test should do a quick check about not receiving a stage
migration signal as a result of calling clock_gettime(). We have to
make sure that we go through the vDSO for reading timestamps, without
incurring any in-band syscall. In the ARM case, switching in-band when
calling clock_gettime() would denote a system clocksource which was
not enabled for user MMIO access, which is bad.
