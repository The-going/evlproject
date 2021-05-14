---
menuTitle: "Running tests"
title: "Testing the installation"
weight: 3
---

EVL comes with a series of tests you can run to make sure the core is
performing correctly on your target system.

## latmus: the litmus test for latency

Without any argument, the program called `latmus` runs a 1Khz sampling
loop, collecting the min, max and average latency values obtained for
an EVL thread running in user-space. This is a timer latency benchmark
which does not require any additional interrupt source beyond the
on-board hardware timer readily available to the kernel. You can also
use this program to [calibrate the EVL core timer]({{< relref
"core/timer-calibration.md" >}}), finding the best gravity values for
this timer.

```
# /usr/evl/bin/latmus
```

{{% notice note %}}
To get this test running, you will need `CONFIG_EVL_LATMUS` to be
enabled in the kernel configuration, and loaded into the kernel under
test if you built it as a dynamic module. For those familiar with
Xenomai 3, this program combines the features of the `latency` and
`autotune` utilities available there into a single executable.
{{% /notice %}}

`latmus` accepts the following arguments, given as short or long
option names:

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
absence of _-i_ and _-k_.
{{% /argument %}}

{{% argument "-r --reset" %}}
Reset the gravity values of the EVL core timer to their factory
defaults. These defaults are statically defined by the EVL
platform code.
{{% /argument %}}

{{% argument "-L --load" %}}
Run a stress load in the background when running the latency
test. This option is enabled by default when calibrating the EVL core
timer using the _-t_ option.
{{% /argument %}}

{{% argument "-N --noload" %}}
Do not run any stress load in the background when running the latency
test. This option can be used to force disable the default setting
when calibrating the EVL core timer using the _-t_ option.
{{% /argument %}}

{{% argument "-q --quiet" %}}
Tame down verbosity of the test to the bare minimum, only the final
latency report will be issued when in effect. Passing this option
requires a timeout to be set with the _-T_ option.
{{% /argument %}}

{{% argument "-b --background" %}}
Run the test in the shell's background. All output is suppressed until
the final latency report.
{{% /argument %}}

{{% argument "-a --mode-abort" %}}
Automatically abort upon unexpected switch to in-band mode of the
sampling thread. This option only makes sense when collecting latency
figures from an EVL thread running in user-space (i.e. _-u_).
{{% /argument %}}

{{% argument "-m --measure" %}}
Run a latency measurement test, as opposed to tuning the core
timer. _-i_, _-k_ and _-u_ can be used to select a specific
measurement context, _-u_ applies otherwise. Latency measurement is
the default mode, in absence of the _-t_ option on the command line.
{{% /argument %}}

{{% argument "-t --tune" %}}
Run a core timer calibration procedure, as opposed to measuring the
latency. _-i_, _-k_ and _-u_ can be used to select a specific tuning
context, all of them are applied in sequence otherwise. See
[below]({{< relref "core/timer-calibration.md" >}}).
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
modifier suffix, as a count of _d_ays, _m_inutes, _h_ours or
_s_econds. In absence of modifier, seconds are assumed.
{{% /argument %}}

{{% argument "-A --maxlat-abort=<maxlat>" %}}
Automatically abort the test whenever the max latency figure observed
exceeds \<maxlat\>.
{{% /argument %}}

{{% argument "-v --verbose=<level>" %}}
Set the verbosity level to \<level\>. Setting 0 is identical to
entering quiet mode with _-q_. Any non-zero value is considered when
tuning the EVL core timer (_-t_ option), to control the amount of
debug information the `latmus` companion driver sends to the kernel
log. Defaults to 1, maximum is 2.
{{% /argument %}}

{{% argument "-l --lines=<count>" %}}
Set the number of result lines per page. In measurement mode (_-m_), a new
result header is output after every \<count\> result lines.
{{% /argument %}}

{{% argument "-g --plot=<file>" %}}
Dump an histogram of the collected latency values to \<file\> in a
format which is easily readable by the `gnuplot` utility.
{{% /argument %}}

{{% argument "-H --histogram=<cells>" %}}
Set the number of cells in the histogram, each cell covers one
microsecond of additional latency from 1 to \<cells\>
microseconds. This value is used only if _-g_ is given on the command
line. Defaults to 200, covering up to 200 microseconds in worst-case
latency, which should never be as high on any target platform with EVL.
{{% /argument %}}

{{% argument "-P --priority=<prio>" %}}
Set the scheduling priority of the sampling thread in the SCHED_FIFO
class.  This option only makes sense when collecting latency figures
or tuning the EVL core timer from an EVL thread context (i.e. _-u_ or
_-k_).  Defaults to 90.
{{% /argument %}}

{{% argument "-c --cpu=<nr>" %}}
Set the CPU affinity of the sampling thread.  This option only makes
sense when collecting latency figures or tuning the EVL core timer
from an EVL thread context (i.e. _-u_ or _-k_).  Defaults to 0.
{{% /argument %}}

## hectic: hammering the EVL context switching machinery

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

```
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

## Unit testing

A series of unit testing programs is produced in `$prefix/tests` as
part of building `libevl`. You should run each of them to make sure
everything is fine.

> Running the EVL unit tests
```
# for f in /usr/evl/tests/*; do echo $f && $f; done
/usr/evl/tests/basic-xbuf
thread tfd=7
xfd=8
write->oob_read: 4
write->oob_read: 2
write->oob_read: 1
write->oob_read: 1
oob_read[0]<-write: 1 => 0x41
oob_read[1]<-write: 1 => 0x42
oob_read[2]<-write: 1 => 0x43
oob_read[3]<-write: 1 => 0x44
oob_read[4]<-write: 1 => 0x45
oob_read[5]<-write: 1 => 0x46
oob_read[6]<-write: 1 => 0x47
oob_read[7]<-write: 1 => 0x48
dup(6) => 9
dup2(6, 9) => 9
peer reading from fd=6
oob_write->read: 2
oob_write->read: 2
oob_write->read: 2
inband[0] => 01
inband[1] => 23
inband[2] => 45
/usr/evl/tests/clock-timer-periodic
/usr/evl/tests/clone-fork-exec
thread has efd=7
exec() ok for pid 6855
/usr/evl/tests/detach-self
thread efd=7
detach ret=0
thread efd=7
detach ret=0
/usr/evl/tests/duplicate-element
/usr/evl/tests/fault
/usr/evl/tests/fpu-preload
/usr/evl/tests/mapfd
file proxy has efd=5
mapfd child reading: mapfd-test
/usr/evl/tests/monitor-pi
/usr/evl/tests/monitor-pp-dynamic
/usr/evl/tests/monitor-pp-lower
/usr/evl/tests/monitor-pp-nested
/usr/evl/tests/monitor-pp-pi
/usr/evl/tests/monitor-pp-raise
/usr/evl/tests/monitor-pp-tryenter
/usr/evl/tests/monitor-pp-weak
/usr/evl/tests/monitor-steal
...
```
