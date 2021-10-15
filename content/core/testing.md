---
menuTitle: "Running tests"
title: "Testing the installation"
weight: 4
---

EVL comes with a series of tests you can run to make sure the core is
performing correctly on your target system.

## Unit testing {#evl-unit-testing}

A series of unit testing programs is produced in `$prefix/tests` as
part of building `libevl`. You should run each of them to make sure
everything is fine. The simplest way to do this is as follows:

> Running the EVL unit tests
```sh
# evl test
duplicate-element: OK
monitor-pp-dynamic: OK
monitor-pi: OK
clone-fork-exec: OK
clock-timer-periodic: OK
poll-close: OK
sem-wait: OK
monitor-pp-raise: OK
monitor-pp-tryenter: OK
heap-torture: OK
monitor-pp-lower: OK
poll-read: OK
monitor-deadlock: OK
monitor-wait-multiple: OK
monitor-event: OK
proxy-eventfd: OK
monitor-flags.eshi: OK
monitor-wait-multiple.eshi: OK
sem-wait.eshi: OK
detach-self.eshi: OK
sem-timedwait.eshi: OK
proxy-pipe.eshi: OK
clock-timer-periodic.eshi: OK
proxy-eventfd.eshi: OK
monitor-event.eshi: OK
heap-torture.eshi: OK
poll-sem.eshi: OK
poll-nested.eshi: OK
sem-close-unblock: OK
monitor-steal: OK
basic-xbuf: OK
simple-clone: OK
monitor-flags: OK
poll-sem: OK
sem-timedwait: OK
mapfd: OK
proxy-pipe: OK
poll-flags: OK
poll-nested: OK
monitor-pp-pi: OK
fault: OK
monitor-pi-deadlock: OK
detach-self: OK
monitor-pp-nested: OK
monitor-pp-weak: OK
stax-lock: OK
fpu-preload: OK
```

A few tests from the test suite may fail in case some kernel support
is missing in order to support them, like:

```sh
sched-quota-accuracy.c:213: FAILED: evl_control_sched(44, &p, &q, test_cpu) (=Operation not supported)
sched-quota-accuracy: no kernel support
```

In the example above, _sched-quota-accuracy_ failed because
[CONFIG_EVL_SCHED_QUOTA]({{< relref "core/build-steps#core-kconfig"
>}}) was not set in the kernel configuration. Likewise,
_sched-tp-accuracy_ requires [CONFIG_EVL_SCHED_TP]({{< relref
"core/build-steps#core-kconfig" >}}) to be enabled in the kernel
configuration.

{{% notice tip %}}
The test loop aborts immediately upon a test failure. You may disable
this behavior by running `evl test -k` (i.e. _keep going_) instead.
{{% /notice %}}

## hectic: hammering the EVL context switching machinery {#hectic-program}

By default, the `hectic` program runs a truckload of EVL threads both
in user and kernel spaces, for exercising the scheduler of the
autonomous core. In addition, this test can specifically stress the
floating-point management code to make sure the FPU is shared
flawlessly between out-of-band and in-band thread contexts.

{{% notice note %}}
To get this test running, you will need `CONFIG_EVL_HECTIC` to be
enabled in the kernel configuration, and loaded into the kernel under
test if you built it as a dynamic module.
{{% /notice %}}

```sh
# /usr/evl/bin/hectic -s 200
== Testing FPU check routines...
== FPU check routines: OK.
== Threads: switcher_ufps0-0 rtk0-1 rtk0-2 rtup0-3 rtup0-4 rtup_ufpp0-5 rtup_ufpp0-6 rtus0-7 rtus0-8 rtus_ufps0-9 rtus_ufps0-10 rtuo0-11 rtuo0-12 rtuo_ufpp0-13 rtuo_ufpp0-14 rtuo_ufps0-15 rtuo_ufps0-16 rtuo_ufpp_ufps0-17 rtuo_ufpp_ufps0-18 fpu_stress_ufps0-19 switcher_ufps1-0 rtk1-1 rtk1-2 rtup1-3 rtup1-4 rtup_ufpp1-5 rtup_ufpp1-6 rtus1-7 rtus1-8 rtus_ufps1-9 rtus_ufps1-10 rtuo1-11 rtuo1-12 rtuo_ufpp1-13 rtuo_ufpp1-14 rtuo_ufps1-15 rtuo_ufps1-16 rtuo_ufpp_ufps1-17 rtuo_ufpp_ufps1-18 fpu_stress_ufps1-19 switcher_ufps2-0 rtk2-1 rtk2-2 rtup2-3 rtup2-4 rtup_ufpp2-5 rtup_ufpp2-6 rtus2-7 rtus2-8 rtus_ufps2-9 rtus_ufps2-10 rtuo2-11 rtuo2-12 rtuo_ufpp2-13 rtuo_ufpp2-14 rtuo_ufps2-15 rtuo_ufps2-16 rtuo_ufpp_ufps2-17 rtuo_ufpp_ufps2-18 fpu_stress_ufps2-19 switcher_ufps3-0 rtk3-1 rtk3-2 rtup3-3 rtup3-4 rtup_ufpp3-5 rtup_ufpp3-6 rtus3-7 rtus3-8 rtus_ufps3-9 rtus_ufps3-10 rtuo3-11 rtuo3-12 rtuo_ufpp3-13 rtuo_ufpp3-14 rtuo_ufps3-15 rtuo_ufps3-16 rtuo_ufpp_ufps3-17 rtuo_ufpp_ufps3-18 fpu_stress_ufps3-19
RTT|  00:00:01
RTH|---------cpu|ctx switches|-------total
RTD|           0|         568|         568
RTD|           3|         853|         853
RTD|           2|         739|         739
RTD|           1|         796|         796
RTD|           0|         627|        1195
RTD|           2|        1258|        1997
RTD|           3|        1197|        2050
RTD|           1|        1311|        2107
RTD|           0|         627|        1822
RTD|           2|        1250|        3247
RTD|           3|        1254|        3304
RTD|           1|        1254|        3361
RTD|           2|        1254|        4501
RTD|           1|        1254|        4615
RTD|           0|         684|        2506
RTD|           3|        1311|        4615
RTD|           3|        1256|        5871
RTD|           2|        1311|        5812
RTD|           0|         684|        3190
RTD|           1|        1311|        5926
...
```

## latmus: the litmus test for latency {#latmus-program}

{{% notice tip %}}
If you plan for measuring the worst case latency on your target
system, you should run the [evl check]({{< relref
"core/commands#evl-check-command" >}}) command on such system in
order to detect any obvious misconfiguration of the kernel early on.
{{% /notice %}}

With the sole `-m` option or without any argument, the
[latmus](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/benchmarks/latmus.c)
application runs a 1Khz sampling loop, collecting the min, max and
average latency values obtained for an EVL thread running in
user-space which responds to [timer events]({{< relref
"core/benchmarks/_index.md#latmus-timer-response-time" >}}). This is a
basic latency benchmark which does not require any additional
interrupt source beyond the on-chip hardware timer readily available
to the kernel.

In addition, you can use this application to measure the response time
of a thread running in user-space to external interrupts, specifically
to [GPIO events]({{< relref
"core/benchmarks/_index.md#latmus-timer-response-time" >}}). This
second call form is selected by the `-Z` and `-z` option switches.

Finally, passing `-t` starts a [calibration of the EVL core timer]({{<
relref "core/runtime-settings.md" >}}), finding the best configuration
values.

{{% notice tip %}}
Unless you only plan to measure [in-band response time to GPIO
events]({{< relref
"core/benchmarks/_index.md#latmus-gpio-response-time" >}}), you will
need `CONFIG_EVL_LATMUS` to be enabled in the kernel configuration to
run the timer calibration or the [response to timer test]({{< relref
"core/benchmarks/_index.md#latmus-timer-response-time" >}}). This
driver must be loaded into the kernel under test if you built it as a
dynamic module. For those familiar with [Xenomai 3 Cobalt](https://git.xenomai.org/xenomai/-/wikis/home), this program
combines and extends the features of the
[latency](https://xenomai.org/documentation/xenomai-3/html/man1/latency/index.html)
and
[autotune](https://xenomai.org/documentation/xenomai-3/html/man1/autotune/index.html)
utilities.
{{% /notice %}}

[latmus](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/benchmarks/latmus.c)
accepts the following arguments, given as short or long option names:

{{% argument "-i --irq" %}}
Collect latency figures or tune the EVL core timer from the context of
an in-kernel interrupt handler.
{{% /argument %}}

{{% argument "-k --kernel" %}}
Collect latency figures or tune the EVL core timer from the context of
a kernel-based EVL thread.
{{% /argument %}}

{{% argument "-u --user" %}}
Collect latency figures or tune the EVL core timer from the context of
an EVL thread running in user-space. This is the default mode, in
absence of `-i` and `-k`.
{{% /argument %}}

{{% argument "-s --sirq" %}}
Measure the delay between the moment a [synthetic interrupt]({{<
relref "dovetail/pipeline/synthetic.md" >}}) is posted from the
out-of-band stage and when it is eventually received by its in-band
handler. When measured under significant workload pressure, this gives
the worst case interrupt latency experienced by the **in-band kernel** due
to local interrupt disabling (i.e. _stalling_ the in-band pipeline
stage). Therefore, this has nothing to do with the much shorter and
bounded interrupt latency observed from the out-of-band stage by EVL
applications.
{{% /argument %}}

{{% argument "-r --reset" %}}
Reset the gravity values of the EVL core timer to their factory
defaults. These defaults are statically defined by the EVL
platform code.
{{% /argument %}}

{{% argument "-q --quiet" %}}
Tame down verbosity of the test to the bare minimum, only the final
latency report will be issued when in effect. Passing this option
requires a timeout to be set with the `-T` option.
{{% /argument %}}

{{% argument "-b --background" %}}
Run the test in the shell's background. All output is suppressed until
the final latency report.
{{% /argument %}}

{{% argument "-K --keep-going" %}}
Keep the execution going upon unexpected switch to in-band mode of
the responder thread. Normally, any [switch to in-band mode]({{< relref
"core/user-api/thread/_index.md#thread-services" >}}) from the thread
responding to timer/GPIO events would cause the execution to stop with
an error message, since the latency figures would be tainted by a
transition to the non real-time context. This option tells `latmus` to
keep going regardless; it only makes sense for debugging purpose, when
collecting latency figures from an EVL thread running in user-space
(i.e. `-u`).
{{% /argument %}}

{{% argument "-m --measure" %}}
Measure the response time to [timer events]({{< relref
"core/benchmarks/_index.md#latmus-timer-response-time" >}}).  In
addition to this option, `-i`, `-k` and `-u` select a specific
measurement context, `-u` applies by default. Measurement of response
time to timer events is the default mode, in absence of the `-t`, `-Z` and `-z`
options on the command line.
{{% /argument %}}

{{% argument "-t --tune" %}}
Run a core timer calibration procedure. `-i`, `-k` and `-u` can be
used to select a specific tuning context, all of them are applied in
sequence otherwise. See [below]({{< relref "core/runtime-settings.md"
>}}). This option is mutually exclusive with `-m`, `-Z` and `-z`.
{{% /argument %}}

{{% argument "-p --period=<µsecs>" %}}
Set the sampling period to \<µsecs\>. By default, 1000 is used (one
tick every millisecond or 1Khz). The slowest sampling period is
1000000 (1Hz).
{{% /argument %}}

{{% argument "-T --timeout=<duration>[dhms]" %}}
The duration of the test, excluding the one second warmup period. This
option enables a timeout which stops the test automatically after the
specified runtime has elapsed. By default, the test runs indefinitely,
or until ^C is pressed. The duration is interpreted according to the
modifier suffix, as a count of **d**ays, **m**inutes, **h**ours or
**s**econds. In absence of modifier, seconds are assumed.
{{% /argument %}}

{{% argument "-A --maxlat-abort=<maxlat>" %}}
Automatically abort the test whenever the max latency figure observed
exceeds \<maxlat\>.
{{% /argument %}}

{{% argument "-v --verbose=<level>" %}}
Set the verbosity level to \<level\>. Setting 0 is identical to
entering quiet mode with `-q`. Any non-zero value is considered when
tuning the EVL core timer (`-t` option), to control the amount of
debug information the `latmus` companion driver sends to the kernel
log. Defaults to 1, maximum is 2.
{{% /argument %}}

{{% argument "-l --lines=<count>" %}}
Set the number of result lines per page. In measurement mode (`-m`), a new
result header is output after every \<count\> result lines.
{{% /argument %}}

{{% argument "-g --plot=<file>" %}}
Dump an histogram of the collected latency values to \<file\> in a
format which is easily readable by the `gnuplot` utility.
{{% /argument %}}

{{% argument "-H --histogram=<cells>" %}}
Set the number of cells in the histogram, each cell covers one
microsecond of additional latency from 1 to \<cells\>
microseconds. This value is used only if `-g` is given on the command
line. Defaults to 200, covering up to 200 microseconds in worst-case
latency, which should never be as high on any target platform with EVL.
{{% /argument %}}

{{% argument "-P --priority=<prio>" %}}
Set the scheduling priority of the responder thread in the SCHED_FIFO
class.  This option only makes sense when collecting latency figures
or tuning the EVL core timer from an EVL thread context (i.e. `-u` or
`-k`).  Defaults to 90.
{{% /argument %}}

{{% argument "-c --cpu=<nr>" %}}
Set the CPU affinity of the responder thread.  This option only makes
sense when collecting latency figures or tuning the EVL core timer
from an EVL thread context (i.e. `-u` or `-k`).  Defaults to 0.
{{% /argument %}}

{{% argument "-Z --oob-gpio=<host>" %}}
Start an out-of-band test measuring the [response time to GPIO
events]({{< relref
"core/benchmarks/_index.md#latmus-gpio-response-time" >}}) from
the out-of-band stage, i.e. relying on real-time capabilities of the EVL
core. The argument is the host name or IPv4 addresses of the remote
board which monitors the response time from the SUT running
the
[latmus](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/benchmarks/latmus.c)
application. This option must be associated with `-I` and `-O` to
specify the GPIO chip(s) and pin numbers to use.
{{% /argument %}}

{{% argument "-z --inband-gpio=<host>" %}}
Start an in-band test measuring the [response time to GPIO events]({{<
relref "core/benchmarks/_index.md#latmus-gpio-response-time" >}}) in
plain in-band mode. The argument is the host name or IPv4 address of
the remote board which monitors the response time from the
SUT running the
[latmus](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/benchmarks/latmus.c)
application. This option must be associated with `-I` and `-O` to
specify the GPIO chip(s) and pin numbers to use.
{{% /argument %}}

{{% argument "-I --gpio-in=<gpiochip-name>,<pin-number>[,rising-edge|falling-edge]" %}}
Specify the GPIO chip and pin number to be used for receiving the [GPIO
pulses]({{< relref
"core/benchmarks/_index.md#latmus-gpio-response-time" >}}) from the
remote monitor board. Optionally, you can select whether GPIO events
should be triggered on the rising edge (default) or falling edges of
GPIO signals. This option only makes sense whenever `-Z` or `-z` are
present on the command line too.
{{% /argument %}}

{{% argument "-O --gpio-out=<gpiochip-name>,<pin-number>" %}}
Specify the GPIO chip and pin number to be used for acknowledging the
[GPIO pulses]({{< relref
"core/benchmarks/_index.md#latmus-gpio-response-time" >}}) received
from the monitor board.
This option only makes sense whenever `-Z` or `-z` are present on the
command line too.
{{% /argument %}}

{{% notice tip %}}
If [latmus]({{< relref "#latmus-program" >}}) fails starting with an
_Invalid argument_ error, double-check the CPU number passed to -c if
given. The designated CPU must be part of the out-of-band CPU set
known to the EVL core. Check this file
_/sys/devices/virtual/evl/control/cpus_ to know which CPUs are part of
this set.
{{% /notice %}}

---

{{<lastmodified>}}
