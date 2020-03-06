---
menuTitle: "Commands"
weight: 5
---

# The 'evl' command

The _evl_ umbrella utility can run the set of (sub)commands available
for controlling, inspecting and testing the state of the EVL
core. Each of these commands is implemented by an external plugin,
which can be a mere executable, or a script in whatever language. The
general syntax is as follows:

```
evl [-V] [-P <cmddir>] [<command> [command-args]]
```

- \<command\> may be either:
  * **ps** which report a snapshot of the current EVL threads
  * **test** which run the EVL tests
  * **trace** which is a simple front-end to the _ftrace_ interface for EVL

- _-P_ switches to a different installation path for command
  plugins, which is located at $prefix/libexec by default.

- _-V_ displays the version information then exits. The information is
  extracted from the `libevl` library the EVL command depends on,
  displayed in the following format:

    **evl.\<serial\> \-\- #\<git-HEAD-commit\> (\<git-HEAD-date\>) [ABI \<revision\>]**

    where **\<serial\>** is the `libevl` serial release number, the
    **\<git-HEAD\>** information refers to the topmost GIT commit
    which is present in the binary distribution the _evl_ command is
    part of, and **\<revision\>** refers to the kernel ABI this binary
    distribution is compatible with. For instance:

```
~ # evl -V
evl.0 -- #ff99204 (2020-01-04 18:34:10 +0100) [ABI 12]
```

{{% notice info %}}
The information following the double dash may be omitted if the built
sources were not version-controlled by GIT.
{{% /notice %}}

Without any argument, the _evl_ utility displays this general help,
along with a short help string for each of the supported (sub)commands
found in **\<cmddir\>**, such as:

```
~ # evl
evl [-V] [-P <cmddir>] [<command> [command-args]], with <command> in:
ps           report a snapshot of the current EVL threads
test         run EVL tests
trace        ftrace control front-end for EVL
```

### Reporting a snapshot of the current EVL threads (ps) {#evl-ps-command}

When you need to know which threads are currently present in your
system, **evl ps** comes in handy. The command syntax - which
supports short and long options formats - is as follows:

```
$ evl ps [-c --cpu=<cpu>[,<cpu>...]]
         [-s --state]
         [-t --times]
         [-p --policy]
         [-l --long]
         [-n --numeric]
         [-S --sort=<key>]
         [-h --help]
```

This command fetches the information it needs from the [/sysfs
attributes]({{< relref
"core/user-api/thread/_index.md#thread-sysfs-data" >}}) the EVL core
exports for every thread it manages. The output is organized in groups
of values, representing either runtime parameters or statistics for
each displayed thread:

- **NAME** reports the thread name, as specified in the
    [evl_attach_self()]({{< relref
    "core/user-api/thread/_index.md#evl_attach_thread" >}}) library
    call.

- **CPU** is the processor id. the thread is currently pinned to.

- **PID** is the process id. inband-wise, since any [EVL thread]({{<
  relref "core/user-api/thread/_index.md" >}}) is originally a regular
  Linux [k]thread. This value belongs to the global namespace
  (i.e. task_pid_nr()).

- **SCHED** is the current [scheduling policy]({{< relref
  "core/user-api/scheduling/_index.md" >}}) for the thread.

- **PRIO** is the priority level in the [scheduling policy]({{< relref
  "core/user-api/scheduling/_index.md" >}}) for the thread.

- **ISW** counts the number of [inband switches]({{< relref
    "core/user-api/thread/_index.md" >}}). Under normal circumstances,
    this count should remain stable over time once the thread has
    entered its work loop. As the only exception, a thread which
    undergoes the [SCHED_WEAK policy]({{< relref
    "core/user-api/scheduling/_index.md#SCHED_WEAK" >}}) may see this
    counter progress as a result of calling out-of-band services. For
    all other scheduling policies, observing any increase in this
    value after the time-critical loop was entered is a sure sign of a
    problem in the application code, which might be calling real-time
    unsafe services when it should not.

- **CTXSW** counts the number of context switches _performed by the
    EVL core_ for the thread. This value is incremented each time the
    thread resumes from preemption or suspension in the EVL
    core. CAUTION: this has nothing to do with the context switches
    performed by the main kernel logic.

- **SYS** is the number of out-of-band system calls issued by the
  thread to the EVL core. Basically, this is the number of
  [out-of-band I/O]({{< relref "core/user-api/io/_index.md" >}})
  requests the thread has issued so far, since [everything is a
  file]({{< relref "core/_index.md#everything-is-a-file" >}}) in the
  EVL core.

- **RWA** counts the number of **R**emote **WA**keup signals the EVL
    core had to send so far for waking up the thread whenever it was
    sleeping on a remote CPU. For instance, this would happen if two
    threads running on different CPUs were to synchronize on an [EVL
    event]({{< relref "core/user-api/event" >}}). Waking up a remote
    thread entails sending an [inter-processor interrupt]({{< relref
    "dovetail/pipeline/pipeline_inject.md#oob-ipi" >}}) to the CPU
    that thread sleeps on for kicking the rescheduling procedure,
    which entails more overhead than a local wakeup. If this counter
    increases like crazy when your application runs, you might want to
    check the situation with respect to CPU affinity, to make sure the
    current distribution of threads over the available CPUs is
    actually what you want.

- **STAT** gives an abbreviated runtime status of the thread as
    follows:

    * 'w'/'W' &#8702; Waits on a resource with/without timeout (TIMEOUT displays the time before timeout)
    * 'D' &#8702; Delayed (TIMEOUT displays the remaining sleep time)
    * 'p' &#8702; Periodic timeline (kthread only, TIMEOUT displays the remaining time until the next period starts)
    * 'R' &#8702; Ready to run (i.e. neither blocked nor suspended, but waiting for the CPU to be available)
    * 'X' &#8702; Running in-band
    * 'T' &#8702; Ptraced and stopped (whenever traced by a debugger such as _gdb[server]_)
    * 'r' &#8702; Undergoes round-robin (when [SCHED_RR]({{< relref
        "core/user-api/scheduling/_index.md#SCHED_RR" >}}) is in effect)
    * 'S' &#8702; Forcibly suspended (cumulative with 'W' state, won't resume until lifted)

- **TIMEOUT** is the remaining time before some timer which was
    started specifically for the thread fires. Which timer was started
    depends on the undergoing operation for such thread, which may
    block until a resource is available, wait for the next period in a
    timeline and so on. See **STAT** for details.

- **%CPU** is the current amount of CPU horsepower consumed by the
     thread over the last second. When an [out-of-band interrupt]({{<
     relref "core/kernel-api/interrupts/_index.md" >}}) preempts a
     thread, the time spent handling it is charged to that thread.

- **CPUTIME** reports the cumulated CPU time already consumed by the
     thread, using a _minutes:milliseconds.microseconds_ format.

The command options allow to select which threads and which data
should be displayed:

- the _-c_ option filters the output on the CPU the threads are
  pinned on. The argument is a comma-separated list of CPU
  numbers. Ranges are also supported via the usual dash separator. For
  instance, the following command would report threads pinned on CPU0,
  and all CPUs from CPU3 to CPU15.

```
$ evl ps -c 0,3-15
```

- _-s_ includes information about the thread state, which is ISW,
  CTXSW, SYS, RWA and STAT.

- _-t_ includes the thread times, which are TIMEOUT, CPU% and CPUTIME.

- _-p_ includes the scheduling policy information, which is SCHED and
  PRIO.

- _-l_ enables the long output format, which is a combination of all
  information groups.

- _-n_ selects a numeric output for the **STAT** field, instead of the
  one-letter flags. This actually dumps the 32-bit value representing
  all aspects of a thread status in the EVL core, which contains more
  information than reported by the abbreviated format. EVL hackers
  want that.

- _-S_ sorts the output according to a given sort key in increasing
  order. The following sort keys are understood:

  * 'c' sorts by **CPU** values
  * 'i' sorts by **ISW** values
  * 't' sorts by **CPUTIME** values
  * 'x' sorts by **CTXSW** values
  * 'w' sorts by **RWA** values
  * 'r' sorts in reverse order (decreasing order)

For instance, the following command would list the times of all
threads from CPU14 by decreasing CPU time consumption:

```
$ evl ps -t -Srt -c14
CPU   PID   TIMEOUT      %CPU   CPUTIME     NAME
 14   2603     -           0.6  00:435.952  rtup_ufpp14-5:2069
 14   2604     -           0.6  00:430.147  rtup_ufpp14-6:2069
 14   2599     -           0.5  00:423.118  rtup14-3:2069
 14   2600     -           0.5  00:420.293  rtup14-4:2069
 14   2595     -           0.3  00:207.143  [rtk1@14:2069]
 14   2597     -           0.3  00:204.301  [rtk2@14:2069]
 14   2619     -           0.2  00:186.139  rtuo_ufpp14-14:2069
 14   2617     -           0.2  00:185.497  rtuo_ufpp14-13:2069
 14   2623     -           0.2  00:184.812  rtuo_ufpp_ufps14-18:2069
 14   2622     -           0.2  00:184.772  rtuo_ufpp_ufps14-17:2069
 14   2621     -           0.2  00:181.692  rtuo_ufps14-16:2069
 14   2616     -           0.2  00:181.329  rtuo14-12:2069
 14   2615     -           0.2  00:181.230  rtuo14-11:2069
 14   2620     -           0.2  00:180.604  rtuo_ufps14-15:2069
 14   2572     -           0.0  00:000.006  rtus_ufps13-9:2069
 14   2125     -           0.0  00:000.006  rtus1-7:2069
 14   2646     -           0.0  00:000.005  rtus15-8:2069
 14   2650     -           0.0  00:000.005  rtus_ufps15-10:2069
 14   2310     -           0.0  00:000.005  rtus6-7:2069
```

- _-h_ displays the command help string.

### Controlling the kernel tracer (trace) {#evl-trace-command}

The **trace** command provides a simple front-end for controlling the
function tracer which is part of the [FTRACE kernel
framework](https://www.kernel.org/doc/Documentation/trace/ftrace.txt),
in a way which is EVL-aware.

In order to use this tracer, make sure to enable the following
features in your [kernel build]({{< relref
"core/build-steps.md#building-evl-core" >}}):

- CONFIG_TRACER_SNAPSHOT
- CONFIG_TRACER_SNAPSHOT_PER_CPU_SWAP
- CONFIG_FUNCTION_TRACER

The command syntax is as follows:

```
evl trace [-e [-s <buffer_size>]]
    	  [-d] [-p] [-f] [-h]
	  [-c <cpu>]
	  [-h]
```

The command options allow for a straightforward use of the function
tracer:

- _-e_ enables the tracer in the kernel. From this point, FTRACE
  starts logging information about a set of kernel functions which may
  be traversed while the system executes. By default, this switch only
  enables tracing for out-of-band IRQ events, CPU idling events, and
  all (in-kernel) EVL core routines. If a particular CPU is mentioned
  with _-c_ along with _-e_, then per-CPU tracing is enabled for
  \<cpu\>.

- if _-f_ is mentioned, all kernel functions traversed in the course
  of execution are logged, not only the minimal subset enabled by
  default by _-e_. CAUTION: enabling full tracing may cause a massive
  overhead.

- _-s_ changes the size of the FTRACE buffer on each tracing CPU to
  \<buffer_size\>. If a particular CPU is mentioned with _-c_ along with
  _-s_, then the change is applied to the snapshot buffer of \<cpu\>
  only.

- _-d_ fully disables the tracer which stops logging events on all
  CPUs.

- _-p_ prints out the contents of the trace buffer. If a particular
  CPU is mentioned with _-c_ along with _-p_, then only the snapshot
  buffer of \<cpu\> is dumped.

- _-h_ displays the command help string.

For instance, the following command starts tracing all kernel
routines:

```
$ evl trace -ef
```

### Running the test suite (test) {#evl-test-command}

This command is a short-hand for running the [EVL test suite]({{<
relref "core/testing.md#evl-unit-testing" >}}). The usage is as
follows:

```
$ evl test [-l][-L][-k] [test-list]
```

With no argument, this command runs all of the tests available from
the default installation path at $prefix/tests:

```
$ evl test
duplicate-element: OK
monitor-pp-dynamic: OK
monitor-pi: OK
clone-fork-exec: OK
...
```

You can also chose to run a specific set of tests by mentioning them
as arguments to the command, such as:

```
$ evl test duplicate-element monitor-pi
duplicate-element: OK
monitor-pi: OK
```

You may ask for listing the available tests instead of executing them,
by using the _-l_ switch:

```
$ evl test -l
duplicate-element
monitor-pp-dynamic
monitor-pi
clone-fork-exec
...
```

In a variant aimed at making scripting easier, you can ask for the
absolute paths instead:

```
$ evl test -L proxy-pipe mapfd
/usr/evl/tests/proxy-pipe
/usr/evl/tests/mapfd
```

If some test goes wrong, the command normally stops
immediately. Passing _-k_ would allow it to keep going until the end
of the series.

### Implementing your own EVL commands {#evl-add-plugin}

You can implement your own command plugins, then install them into the
$prefix/libexec directory so that the _evl_ utility can find
them. This utility sets a few environment variables before calling any
plugin. Your plugin code can retrieve them using
[getenv(3)](http://man7.org/linux/man-pages/man3/getenv.3.html) from a
C program, or directly dereference those variables from a shell:

| Variable | Description | Default value |
| -------- | ----------- | ------------- |
| EVL_CMDDIR | Where to find the plugins | $prefix/libexec (*) |
| EVL_TESTDIR | Where to find the tests | $prefix/tests |
| EVL_SYSDIR | root of the /sysfs hierarchy for EVL devices | /sys/devices/virtual |
| EVL_TRACEDIR | root of ftrace hierarchy | /sys/kernel/debug/tracing |

(*) may be overriden using the _-P_ option.

---

{{<lastmodified>}}
