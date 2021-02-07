---
menuTitle: "Commands"
weight: 5
---

# The 'evl' command

The 'evl' umbrella utility can run the set of base commands available
for controlling, inspecting and testing the state of the EVL core and
any command matching the 'evl-*' glob pattern which may be reachable
from the shell $PATH variable. The way the 'evl' utility centralizes
access to a variety of EVL-related commands is very similar to that of
[git](https://git-scm.com/) on purpose. Each of the EVL commands is
implemented by an external plugin, which can be a mere executable, or
a script in whatever language. The only requirement is that the caller
must have execute permission on such file to run it.

The general syntax is as follows:

```
evl [-V] [-P <cmddir>] [-h] [<command> [command-args]]
```

- \<command\> may be any command word listed by 'evl -h', such as:
  **check**        which checks a kernel configuration for common issues
  **ps**           which reports a snapshot of the current EVL threads
  **test**         for running the EVL test suite
  **trace**        which is a simple front-end to the _ftrace_ interface for EVL

- `-P` switches to a different installation path for base command
  plugins, which is located at $prefix/libexec by default.

- `-V` displays the version information then exits. The information is
  extracted from the `libevl` library the EVL command depends on,
  displayed in the following format:

    **evl.\<serial\> \-\- #\<git-HEAD-commit\> (\<git-HEAD-date\>) [ABI \<revision\>]**

    where **\<serial\>** is the `libevl` serial release number, the
    **\<git-HEAD\>** information refers to the topmost GIT commit
    which is present in the binary distribution the 'evl' command is
    part of, and **\<revision\>** refers to the kernel ABI this binary
    distribution is compatible with. For instance:

```
~ # evl -V
evl.0 -- #1c6115c (2020-03-06 16:24:00 +0100) [requires ABI 19]
```

{{% notice info %}}
The information following the double dash may be omitted if the built
sources were not version-controlled by GIT.
{{% /notice %}}

- given only `-h` or without any argument, 'evl' displays this general
help, along with a short help string for each of the supported
commands found in **\<cmddir\>**, such as:

```
~ # evl
usage: evl [options] [<command> [<args>]]
-P --prefix=<path>   set command path prefix
-V --version         print library and required ABI versions
-h --help            this help

available commands:

check        check kernel configuration
gdb          debug EVL command plugin with GDB
ps           report a snapshot of the current EVL threads
test         run EVL tests
trace        ftrace control front-end for EVL
```

### Checking a kernel configuration (check) {#evl-check-command}

`evl check` may be the very first `evl` command you should run from a
newly installed target system which is going to run the EVL core. This
command checks a kernel configuration for common issues which may
increase latency. The general syntax is as follows:

```
$ evl check [-f --file=<.config>]
      	    [-L --check-list=<file>]
      	    [-a --arch=<cpuarch>]
	    [-H --hash-size=<N>]
	    [-q --quiet]
	    [-h --help]
```

The kernel configuration to verify is a regular `.config` file which
contains all the settings for building a kernel image. If none is
specified using the `-f` option, the command defaults to reading
`/proc/config.gz` on the current machine. If this fails because any of
`CONFIG_IKCONFIG` or `CONFIG_IKCONFIG_PROC` was disabled in the
running kernel, the command fails.

The check list contains a series of single-line assertions which are
tested against the contents of the kernel configuration. You can
override the default check list stored at
`$prefix/libexec/kconf-checklist.evl` with our own set of checks with
the `-L` option.  Each assertion follows the BNF-like syntax below:

```
assertion   : expr conditions
            | "!" expr conditions

expr        : symbol /* matches =y and =m */
            | symbol "=" tristate

tristate  : "y"
          | "m"
          | "n"

conditions  : dependency
            | dependency arch

dependency  : "if" symbol       /* true if set as y/m */

arch        : "on" cputype

cputype     : $(uname -m)
```

For instance:

- _CONFIG\_FOO must be set whenever CONFIG\_BAR is unset_ can
be written as `CONFIG_FOO if !CONFIG_BAR`.

- _CONFIG\_FOO must not be set_ can be written as `!CONFIG_FOO`, or
  conversely `CONFIG_FOO=n`.

- _CONFIG\_FOO must be built as module on aarch32 or aarch64_ can be
written as `CONFIG_FOO=m on aarch`.

- _CONFIG\_FOO must not be built-in on aarch64 if CONFIG\_BAR is set_
can be written as `!CONFIG_FOO=y if CONFIG_BAR on aarch`.

Assertions in the check list may apply to a particular CPU
architecture. Normally, the command should be able to figure out which
architecture the kernel configuration file applies to by inspecting
the first lines, looking for the "Linux/<arch>" pattern. However, you
might have to specify this information manually to the command using
the `-a` option if the file referred to by the `-f` option does not
contain such information. The architecture name (_cputype_) should
match the output of $(uname -m) or some abbreviated portion of
it. However, _arm64_ and _arm_ are automatically translated to
_aarch64_ and _aarch32_ when found in an assertion or passed to the
`-a` option.

The [default check list](https://git.evlproject.org/libevl.git/tree/utils/kconf-checklist.evl)
translates the configuration-related information gathered in the [caveat section]({{< relref
"core/caveat.md" >}}) as follows:
```
CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y if CONFIG_CPU_FREQ
CONFIG_DEBUG_HARD_LOCKS=n
CONFIG_ACPI_PROCESSOR_IDLE=n
CONFIG_LOCKDEP=n
CONFIG_DEBUG_LIST=n
CONFIG_DEBUG_VM=n
CONFIG_DEBUG_PER_CPU_MAPS=n
CONFIG_KASAN=n
```

The command returns the following information:

- the wrong settings detected in the kernel configuration are written
  to stdout, unless the quiet `-q` option was given.

- the number of failed assertions is returned via the shell exit
  code ($?).

> Example: checking the current kernel configuration
```
~ # evl check
CONFIG_ACPI_PROCESSOR_IDLE=y
~ # echo $?
1
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
    "dovetail/pipeline/pipeline_inject#oob-ipi" >}}) to the CPU
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

- the `-c` option filters the output on the CPU the threads are
  pinned on. The argument is a comma-separated list of CPU
  numbers. Ranges are also supported via the usual dash separator. For
  instance, the following command would report threads pinned on CPU0,
  and all CPUs from CPU3 to CPU15.

```
$ evl ps -c 0,3-15
```

- `-s` includes information about the thread state, which is ISW,
  CTXSW, SYS, RWA and STAT.

- `-t` includes the thread times, which are TIMEOUT, CPU% and CPUTIME.

- `-p` includes the scheduling policy information, which is SCHED and
  PRIO.

- `-l` enables the long output format, which is a combination of all
  information groups.

- `-n` selects a numeric output for the **STAT** field, instead of the
  one-letter flags. This actually dumps the 32-bit value representing
  all aspects of a thread status in the EVL core, which contains more
  information than reported by the abbreviated format. EVL hackers
  want that.

- `-S` sorts the output according to a given sort key in increasing
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

- `-h` displays the command help string.

### Controlling the kernel tracer (trace) {#evl-trace-command}

The **trace** command provides a simple front-end for controlling the
function tracer which is part of the [FTRACE kernel
framework](https://www.kernel.org/doc/Documentation/trace/ftrace.txt),
in a way which is Dovetail-aware. We typically use this tracer to
analyze high latency spots during the course of the [latmus]({{<
relref "core/testing.md#latmus-program" >}}) program execution.

{{% notice info %}}
There is no Dovetail (or EVL-specific) tracer. Latency spots can be analyzed using
the common kernel _function_ tracer, which reports additional information about
the current execution stage and interrupt state. Trace snapshots are automatically taken
at appropriate times by the [latmus]({{< relref "core/testing.md#latmus-program" >}})
utility in order to help in such analysis.
{{% /notice %}}

In order to use this tracer, make sure to enable the following
features in your [kernel build]({{< relref
"core/build-steps#building-evl-core" >}}):

- CONFIG_TRACER_SNAPSHOT
- CONFIG_TRACER_SNAPSHOT_PER_CPU_SWAP
- CONFIG_FUNCTION_TRACER

The command syntax is as follows:

```
evl trace [-e[<trace_group>][-E<tracepoint_file>][-s<buffer_size>][-t]]
    	  [-d] [-p] [-f] [-h]
	  [-c <cpu>]
	  [-h]
```

{{% notice tip %}}
Arguments to options must immediately follow the option letter, without any
spacing in between.
{{% /notice %}}

The command options allow for a straightforward use of the function
tracer:

- `-e` enables the tracer in the kernel, optionally turning on a trace
  group, which is a set of pre-defined tracepoints. From this point,
  FTRACE starts logging information about a set of kernel tracepoints
  which may be traversed while the system executes.

  Up to libevl **r24**, this option-less switch enables tracing for
  out-of-band IRQ events, CPU idling events, and all (in-kernel) EVL
  core routines.

  Since libevl **r25**, the name of a trace group can be mentioned
  right after the option letter, which refers to a pre-defined set of
  tracepoints. Those tracepoints are listed in a separate file which
  should be stored at $EVL_CMDDIR/trace.$name. A tracepoint in such
  file is specified relative to FTRACE's `tracing/` hierarchy, such as
  `irq/irq_pipeline_entry`, which would refer to
  `$EVL_TRACEDIR/tracing/irq/irq_pipeline_entry`. Without argument,
  `-e` behaves as `-e -f`, which enables all kernel tracepoints (see
  `-f`).

```
~# evl trace -eirq
tracing enabled

~ # cat /usr/evl/libexec/trace.irq 
irq/irq_pipeline_entry
irq/irq_pipeline_exit
irq/irq_handler_entry
irq/irq_handler_exit
evl/evl_timer_shot
evl/evl_trigger
evl/evl_latspot
```

  You can either extend the set of pre-defined trace groups by adding
  your own sets to $EVL_CMDDIR, or use the `-E` option to specify an
  arbitray tracepoint file.

  If a particular CPU is mentioned with `-c` along with `-e`, then
  per-CPU tracing is enabled for \<cpu\>.

- `-E` is similar to `-e`, except that its argument refers to an
  arbitrary tracepoint file. This is handy for working with your own
  custom set of tracepoints.

  If a tracepoint listed in the file is invalid, it is silently
  ignored.

```
~# cat > /tmp/custom_traces
evl/evl_schedule
evl/evl_pick_next
evl/evl_switch_context
evl/evl_switch_tail
evl/evl_finish_wait
^D

~# evl trace -E/tmp/custom_traces
tracing enabled
```

- `-t` turns on the _dry run_ mode for `-e` and `-E`, meaning that all
  commands enabling tracepoints are echoed to the output but not
  actually applied. This is a quick way to check the sanity of a
  (custom) tracepoint file.

- if `-f` is mentioned, all kernel functions traversed in the course
  of execution are logged.  CAUTION: enabling full tracing may cause a
  massive overhead.

- `-s` changes the size of the FTRACE buffer on each tracing CPU to
  \<buffer_size\>. If a particular CPU is mentioned with `-c` along with
  `-s`, then the change is applied to the snapshot buffer of \<cpu\>
  only.

- `-d` fully disables the tracer which stops logging events on all
  CPUs.

- `-p` prints out the contents of the trace buffer. If a particular
  CPU is mentioned with `-c` along with `-p`, then only the snapshot
  buffer of \<cpu\> is dumped.

- `-h` displays the command help string.

For instance, the following command starts tracing all kernel
routines:

```
$ evl trace -ef
```

#### Interpreting the Dovetail-specific trace information

The Dovetail-specific information is about:

- whether the in-band stage is stalled and/or irqs are disabled in the
  CPU. 'd' appears in the entry state flags if the in-band stage is
  stalled while hard irqs are enabled in the CPU, 'D' denotes an
  unstalled in-band stage with hard irqs off in the CPU, and '*'
  denotes a combined stalled in-band stage and hard irqs off in the
  CPU.

- whether we are running on the out-of-band stage, if '~' appears in
  the entry flags.

{{% notice tip %}}
You may want to read [this document]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}) for details on
the notion of interrupt stage Dovetail implements.
{{% /notice %}}

For instance:

```
/* hard irqs off, running in-band */
<...>-4164  [003] D...   122.047972: do_syscall_64 <-entry_SYSCALL_64_after_hwframe
/* in-band stalled and hard irqs off, running out-of-band */
<...>-4164  [003] *.~.   122.048021: __evl_schedule <-run_oob_call
/* in-band stalled, hard irqs on, running in-band */
<...>-4164  [003] d...   122.048082: rcu_lockdep_current_cpu_online <-rcu_read_lock_sched_held
```

In addition to this basic information, [latmus]({{< relref
"core/testing.md#latmus-program" >}}) emits a special tracepoint named
_evl\_latspot_ in the trace event log before taking a trace snapshot,
each time the observed maximum latency increases. The frozen trace is
visible in the corresponding per-CPU snapshot buffer. From that point,
you may be able to backtrack to the source(s) of the extra latency. A
typical debug session would look like this:

```
~ # evl trace -ef
tracing enabled

~ # latmus
warming up on CPU1...
RTT|  00:00:01  (user, 1000 us period, priority 98, CPU1)
RTH|----lat min|----lat avg|----lat max|-overrun|---msw|---lat best|--lat worst
RTD|     26.675|     26.951|     27.826|       0|     0|     26.675|     27.826
RTD|     26.712|     27.067|     31.204|       0|     0|     26.675|     31.204
RTD|     26.653|     26.961|     29.160|       0|     0|     26.653|     31.204
RTD|     26.678|     27.067|     29.285|       0|     0|     26.653|     31.204
RTD|     26.759|     27.051|     29.542|       0|     0|     26.653|     31.204
RTD|     26.770|     27.079|     29.266|       0|     0|     26.653|     31.204
^C---|-----------|-----------|-----------|--------|------|-------------------------
RTS|     10.119|     27.029|     31.204|       0|     0|    00:00:06/00:00:06

~ # evl trace -c 1
 ...
          <idle>-0     [001] *.~.   135.363256: do_trace_write_msr <-__switch_to
          <idle>-0     [001] *.~.   135.363256: write_msr: c0000100, value 7ff90973e700
 timer-responder-234   [001] *.~.   135.363256: switch_fpu_return <-dovetail_context_switch
 timer-responder-234   [001] *.~.   135.363257: do_raw_spin_unlock <-__evl_schedule
 timer-responder-234   [001] *.~.   135.363257: do_raw_spin_lock <-evl_wait_schedule
 timer-responder-234   [001] *.~.   135.363258: do_raw_spin_unlock <-evl_wait_schedule
 timer-responder-234   [001] *.~.   135.363258: do_raw_spin_lock <-latmus_oob_ioctl
 timer-responder-234   [001] *.~.   135.363258: do_raw_spin_unlock <-latmus_oob_ioctl
 timer-responder-234   [001] d.~.   135.363259: evl_oob_sysexit: result=0
 timer-responder-234   [001] d.~.   135.363262: pipeline_syscall <-do_syscall_64
 timer-responder-234   [001] d.~.   135.363262: handle_oob_syscall <-pipeline_syscall
 timer-responder-234   [001] d.~.   135.363263: do_oob_syscall <-handle_oob_syscall
 timer-responder-234   [001] d.~.   135.363263: evl_oob_sysentry: syscall=oob_ioctl
 timer-responder-234   [001] d.~.   135.363264: EVL_ioctl <-do_oob_syscall
 timer-responder-234   [001] d.~.   135.363264: evl_get_file <-EVL_ioctl
 timer-responder-234   [001] *.~.   135.363264: do_raw_spin_lock <-evl_get_file
 timer-responder-234   [001] *.~.   135.363265: do_raw_spin_unlock <-evl_get_file
 timer-responder-234   [001] d.~.   135.363265: latmus_oob_ioctl <-EVL_ioctl
 timer-responder-234   [001] d.~.   135.363266: add_measurement_sample <-latmus_oob_ioctl
 timer-responder-234   [001] d.~.   135.363266: evl_latspot: ** latency peak: 31.204 us **
```

In the [latmus]({{< relref "core/testing.md#latmus-program" >}}) case,
part of this analysis would include estimating the delay between the
latest tick date programmed in the hardware and the actual receipt of
the timer interrupt. When tracing is enabled, this information is
automatically produced in the trace log:

```
/* This is when the timer chip is programmed for the next tick. */
<idle>-0     [001] *.~.   135.362244: evl_timer_shot: latmus_pulse_handler at 135.363228 (delay: 984 us, 196195 cycles
...
/* This is when the corresponding timer interrupt is received by Dovetail. */
<idle>-0     [001] *.~.   135.363233: irq_handler_entry: irq=4354 name=Out-of-band LAPIC timer interrupt
```
	  
### Running the test suite (test) {#evl-test-command}

This command is a short-hand for running the [EVL test suite]({{<
relref "core/testing#evl-unit-testing" >}}). The usage is as
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
by using the `-l` switch:

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
immediately. Passing `-k` would allow it to keep going until the end
of the series.

### Implementing your own EVL commands {#evl-add-plugin}

You can implement your own 'evl' command plugins, which may be located
anywhere provided it is reachable from the shell PATH variable with
the proper execute permission bit set. EVL comes with a set of base
plugins available from $prefix/libexec (*).  The latter directory is
implicitly searched for the command _after_ the PATH variable was
considered, which means that you may override any base command with
your own implementation whenever you see fit.

> Crash course: adding the 'foo' command script to ~/tools
```
$ mkdir ~/tools
$ cat > ~/tools/evl-foo
#! /bin/sh
echo "this is your 'evl foo' command"
^D
$ chmod +x ~/tools/evl-foo
$ export PATH=$PATH:~/tools
$ evl foo
this is your 'evl foo' command
```

In addition, 'evl' sets a few environment variables before calling a
plugin. Your plugin executable/script can retrieve them using
[getenv(3)](http://man7.org/linux/man-pages/man3/getenv.3.html) from a
C program, or directly dereference those variables from a shell:

| Variable | Description | Default value |
| -------- | ----------- | ------------- |
| EVL_CMDDIR | Where to find the base plugins | $prefix/libexec |
| EVL_TESTDIR | Where to find the tests | $prefix/tests |
| EVL_SYSDIR | root of the /sysfs hierarchy for EVL devices | /sys/devices/virtual |
| EVL_TRACEDIR | root of ftrace hierarchy | /sys/kernel/debug/tracing |

(*) may be overriden using the `-P` option.

---

{{<lastmodified>}}
