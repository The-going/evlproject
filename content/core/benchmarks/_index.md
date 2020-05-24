---
menuTitle: "Benchmarking"
title: "Running benchmarks"
weight: 6
---

### Measuring response time to interrupts {#measuring-irq-response-time}

Since the real-time infrastructure has to deliver reliable response
times to external events - as in _strictly bounded_ - whatever the
system may be running when they happen, we have to measure the jitter
between the ideal delivery date of such event, and the actual moment
the application starts processing it. With Linux running on the
hardware, reliable means that an upper bound to such jitter can be
determined, although we are using a non-formal, probabilistic method
through countless hours of testing under a significant stress
load. Although the outcome of such test is not by itself
representative of the overall capability of a system to support
real-time applications, such test going wrong would clearly be a
showstopper. No doubt that every real-time infrastructure out there
wants to shine on that one. To measure the response time to interrupts
in different contexts, EVL provides the [latmus]({{< relref
"core/testing.md##latmus-program" >}}) utility. The following figure
illustrates the potential delays which may exist between the moment an
interrupt request is raised by a device, and the time a responder
thread running in the application space can act upon it:

![Alt text](/images/irq-response-measurement.png "Response time to IRQ")

<sup>(1)</sup> There may be multiple causes for the hardware-induced
delay such as (but not limited to):

- some devices may temporarily block the CPU from completing I/O
  transactions which may lead to a stall. For instance, some GPUs
  preventing CPUs to access their I/O memory for a short while, or
  burst mode DMA affecting the execution rate of instructions in the
  CPU by keeping it off the memory bus during transfers.

- the CPU needs to synchronize with the interrupt request internally,
  the instruction pipeline may be affected by the operations involved
  in taking an interrupt, leading to additional latency.

- The time interrupts are masked in the CPU upon request from the
  software, effectively preventing it to take IRQs.

<sup>(2)</sup> Although handling the IRQ in some service routine from
the real-time core is (hopefully) a short process which ends up
readying the responder thread in the scheduler, it may be further
delayed:

- I/D memory caches may have been dirtied by non real-time activities
  while the real-time system was waiting for the next event.  This
  increases the level of cache misses for both code and data, and
  therefore slows down the execution of the real-time infrastructure
  as a whole when it starts handling the incoming event.

- the time needed to handle a write miss is longer when the
  _write-allocate policy_ is enabled in the cache controller, making
  the execution slower when this happens.

<sup>(3)</sup> Eventually, the scheduler is called  in order to
reconsider which thread should run on the CPU the responder thread was
sleeping on when readied, which is subject to more potential delays:

- the CPU receiving the IRQ might not be the one the responder sleeps
  on, in which case the former must send a rescheduling request to the
  latter, so that it will resume the responder. This is usually done
  via an inter-processor interrupt, aka [IPI]({{%relref
  "dovetail/porting/arch.md#dealing-with-ipis" %}}). The time required
  for the IPI to flow to the remote CPU and be handled there further
  extends the delay. A trivial work around would involve setting the
  [IRQ
  affinity](https://www.kernel.org/doc/Documentation/IRQ-affinity.txt)
  to the same CPU the responder runs on, but this may not be possible
  unless the interrupt controller does allow this on your SoC.

- once the scheduler code has determined that the responder thread
  should resume execution, the context switch code is performed to
  restore the memory context and the register file for that thread to
  resume where it left off. Switching memory context may be a lenghty
  operation on some architectures as this affects the caches and
  requires strong synchronization.

- the hardware-induced slowdowns mentioned for step <sup>(2)</sup>
  apply as well.

{{% notice info %}}
Getting into these issues is not a matter of following a dual kernel
vs native preemption approach: all of them bite the same way
regardless.
{{% /notice %}}

### Measuring response time to timer events {#latmus-timer-response-time}

A real-time system normally comes with a way to measure the latency of
its threads on timer events.
[Xenomai](https://xenomai.org/documentation/xenomai-3/html/man1/latency/)
provides one, the
[PREEMPT_RT](https://rt.wiki.kernel.org/index.php/Cyclictest) project
as well, so does EVL with the [latmus]({{< relref
"core/testing.md#latmus-program" >}}) program. No wonder why, running
precisely timed work loops is a basic requirement for real-time
applications. Besides, such a test requires only little preparation:
we don't need any external event source for running it, no specific
equipment is needed, the on-chip high-precision clock timer of the
system under test should be sufficient, therefore we can measure
easily the latency directly from there.

EVL implements this test with the help of the [latmus
driver](https://git.evlproject.org/linux-evl.git/tree/drivers/evl/latmus.c?h=evl/master)
which sends a wake up event to an [EVL-enabled responder thread]({{<
relref "core/thread/_index.md" >}}) created by the [latmus
application]({{< relref "core/testing.md#latmus-program" >}}) each
time a new timer interrupt occurs. The responder thread gets a
timestamp from EVL's [monotonic clock]({{< relref
"core/user-api/clock/_index.md#builtin-clocks" >}}) immediately when
it resumes upon wake up, then sends that information to the driver,
which in turn calculates the latency value, i.e. the delay between the
ideal time the interrupt should have been received and the actual wake
up time reported by the responder thread. This way, we account for all
of the delays [mentioned earlier]({{< relref
"#measuring-irq-response-time" >}}) which might affect the accuracy of
a request for a timed wake up.

The driver accumulates these results, sending an intermediate summary
every second to a logger thread with the minimum, maximum and average
latency values observed over this period. The 1Hz display loop which
is visible while the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) is running is synchronized on
the receipt of such summary. The following figure illustrates this
execution flow:

![Alt text](/images/timer-response-test.png "Response time to timer events")

When the test completes, the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) determines the minimum, worst
case and average latency values over the whole test duration. Upon
request by passing the `-g` option, the [latmus application]({{<
relref "core/testing.md#latmus-program" >}}) dumps an histogram
showing the frequency distribution of the worst case figures which
have been observed over time. The output format can be parsed by
[gnuplot](http://gnuplot.info).

#### Running the timer-based test

First, we need the [latmus
driver](https://git.evlproject.org/linux-evl.git/tree/drivers/evl/latmus.c?h=evl/master)
to be loaded into the kernel for the SUT. Therefore
`CONFIG_EVL_LATMUS` should be enabled in the kernel
configuration. From the command line, the entire test is controlled by
the [latmus application]({{< relref "core/testing.md#latmus-program"
>}}) using the `-m` option, which can be omitted since measuring the
response time to timer events is the default test.

> Measuring response time to timer events
```
# latmus
warming up on CPU1...
RTT|  00:00:01  (user, 1000 us period, priority 90, CPU1)
RTH|----lat min|----lat avg|----lat max|-overrun|---msw|---lat best|--lat worst
RTD|      1.211|      1.325|      2.476|       0|     0|      1.211|      2.476
RTD|      1.182|      1.302|      3.899|       0|     0|      1.182|      3.899
RTD|      1.189|      1.314|      2.486|       0|     0|      1.182|      3.899
RTD|      1.201|      1.315|      2.510|       0|     0|      1.182|      3.899
RTD|      1.192|      1.329|      2.457|       0|     0|      1.182|      3.899
RTD|      1.183|      1.307|      2.418|       0|     0|      1.182|      3.899
RTD|      1.206|      1.318|      2.375|       0|     0|      1.182|      3.899
RTD|      1.206|      1.316|      2.418|       0|     0|      1.182|      3.899
^C
---|-----------|-----------|-----------|--------|------|-------------------------
RTS|      1.182|      1.316|      3.899|       0|     0|    00:00:08/00:00:08
```

> Collecting plottable histogram data (timer test)
```
warming up on CPU1...
RTT|  00:00:01  (user, 1000 us period, priority 90, CPU1)
RTH|----lat min|----lat avg|----lat max|-overrun|---msw|---lat best|--lat worst
RTD|      1.156|      1.273|      1.786|       0|     0|      1.156|      1.786
RTD|      1.170|      1.288|      4.188|       0|     0|      1.156|      4.188
RTD|      1.135|      1.253|      3.175|       0|     0|      1.135|      4.188
RTD|      1.158|      1.275|      2.974|       0|     0|      1.135|      4.188
...
^C
# test started on: Fri Jan 24 15:36:34 2020
# Linux version 5.5.0-rc7+ (rpm@cobalt) (gcc version 9.2.1 20190827 (Red Hat 9.2.1-1) (GCC)) #45 SMP PREEMPT IRQPIPE Wed Jan 22 12:24:03 CET 2020
# BOOT_IMAGE=(tftp)/tqmxe39/switch/bzImage rw ip=dhcp root=/dev/nfs nfsroot=192.168.3.1:/var/lab/tftpboot/tqmxe39/switch/rootfs,tcp,nfsvers=3 nmi_watchdog=0 console=ttyS0,115200 isolcpus=1 evl.oob_cpus=1
# libevl version: evl.0 -- #a3ceb80 (2020-01-22 11:57:11 +0100)
# sampling period: 1000 microseconds
# clock gravity: 2000i 3500k 3500u
# clocksource: tsc
# vDSO access: architected
# context: user
# thread priority: 90
# thread affinity: CPU1
# C-state restricted
# duration (hhmmss): 00:01:12
# peak (hhmmss): 00:00:47
# min latency: 0.205
# avg latency: 3.097
# max latency: 25.510
0 95
1 25561
2 18747
3 2677
4 6592
5 17056
6 664
7 48
8 18
9 12
10 18
11 24
12 14
13 2
14 6
15 5
16 5
17 6
18 23
19 17
20 3
21 1
22 1
23 1
24 1
25 1
```

The output format starts with a comment section which gives specifics
about the test environment and the overall results (all lines from
this section begin with a hash sign). The comment section is followed
by the frequency distribution forming the histogram, in the form of a
series of value pairs: <latency-µs> <number-of-occurrences>. For
instance, "1 25561" means that 25561 wake ups were delayed between 1
(inclusive) and 2 (exclusive) microseconds from the ideal time. A plus
(+) sign appearing after the last count of occurrences means that
there are outliers beyond the limit of the histogram size. In such
event, raising the numnber of cells with the [--histogram=<cells>
option]({{< relref "core/testing.md" >}}) may be a good idea.

#### Interpreting the comment section of a data distribution

- _\# test started on: Fri Jan 24 15:36:34 2020_

    Date the test was started (no kidding).

- _\# Linux version 5.5.0-rc7+ (rpm@cobalt) (gcc version 9.2.1 20190827 (Red Hat 9.2.1-1) (GCC)) #45 SMP PREEMPT IRQPIPE Wed Jan 22 12:24:03 CET 2020_

    The output of [uname -a](http://man7.org/linux/man-pages/man1/uname.1.html) on the system under test.

- _\# BOOT\_IMAGE=(tftp)/tqmxe39/switch/bzImage rw ip=dhcp root=/dev/nfs nfsroot=192.168.3.1:/var/lab/tftpboot/tqmxe39/switch/rootfs,tcp,nfsvers=3 nmi\_watchdog=0 console=ttyS0,115200 isolcpus=1 evl.oob\_cpus=1_

    The kernel command line as returned by /proc/cmdline.

- _\# libevl version: evl.0 -- #a3ceb80 (2020-01-22 11:57:11 +0100)_

    The version information extracted from [libevl]({{< relref
    "core/user-api/_index.md#evl-application" >}}) including its major
    version number, and optionally the [GIT](https://git-scm.com/)
    commit hash and date thereof [libevl]({{< relref
    "core/user-api/_index.md#evl-application" >}}) was built from.

- _\# sampling period: 1000 microseconds_

    The frequency of the event to be responded to by the system under
    test, which can either be a timer tick or a GPIO signal.

- _\# clock gravity: 2000i 3500k 3500u_

    The [calibration settings](({{< relref "core/runtime-settings.md"
    >}})) of the EVL core clock which applied during the test.

- _\# clocksource: tsc_

    The name of the [kernel clock source]({{< relref
    "dovetail/porting/clocksource.md" >}}) used by the EVL core for
    reading timestamps. This value depends on the processor
    architecture, _tsc_ commonly refers to x86.

- _\# vDSO access: architected_

    Since kernel v5.5, the core reports the type of access the EVL
    applications have to the [clock source]({{< relref
    "dovetail/porting/clocksource.md" >}}) via the vDSO. The following
    values are defined:

  - _architected_ denotes a fast (syscall-less) vDSO access to a
    built-in clock source defined by the architecture itself. This is
    the best case.

  - _mmio_ denotes a fast (syscall-less) vDSO access to a [clock source]
    ({{< relref
    "dovetail/porting/clocksource.md#generic-clocksource-vdso" >}})
    exported via Dovetail's generic access to MMIO-based devices. This
    is the second best case.

  - _none_ denotes a not-so-fast access to the kernel clock source
    without vDSO support, which is one of the possible [issues with
    legacy x86 hardware]({{< relref "core/caveat.md#x86-caveat" >}}).
    You could also have such value due to an incomplete port of
    Dovetail to your target system, which may be missing the
    [conversion of existing MMIO clock source]({{< relref
    "dovetail/porting/clocksource.md#generic-clocksource-vdso" >}}) to
    a user-mappable one visible from the generic vDSO mechanism.

- _\# context: user_

    Which was the context of the responder, among:

    - _user_ for EVL threads running in user-space waiting for timer
      events,

    - _kernel_ for EVL kernel threads waiting for timer events,

    - _irq_ for EVL interrupt handlers receiving timer events,

    - _oob-gpio_ for EVL threads running in user-space waiting for
      GPIO events,

    - _inband-gpio_ for regular (non-EVL) threads running in
      user-space waiting for GPIO events.

- _\# thread priority: 90_

  The priority of the responder thread in the out-of-band
[SCHED_FIFO]({{< relref "core/user-api/scheduling#SCHED_FIFO" >}})
class. By definition, any legit value starting from 1 and on gives the
responder thread higher priority than any in-band task running in the
system, including kernel threads of any sort.

- _\# thread affinity: CPU1_

  The processor affinity of the responder thread. If the CPU id. is
suffixed by **-noisol**, then the responder was not running on an
isolated processor during the test, which most likely entailed higher
latency values compared to isolating this CPU - unless you did not
[stress]({{< relref "#stress-load" >}}) the SUT enough or at all while
measuring, which would make the figures obtained quite flawed and
probably worthless anyway.

- _\# C-state restricted_

  Whether the processor C-State was restricted to the shallowest
level, preventing it to enter deeper sleep states which are known to
induce extra latency.

- _\# duration (hhmmss): 00:01:12_

  How long the test ran (according to the wall clock).

- _\# peak (hhmmss): 00:00:47_

  When the worst case value was observed, relatively to the beginning of the test.

- _\# min latency: 0.205_

  The best/shortest latency value observed throughout the test.

- _\# avg latency: 3.097_

  The average latency value observed, accounting for all measurement
  samples collected during the test.

- _\# max latency: 25.510_

  The maximum value observed throughout the test. This is the **worst
  case** value we should definitely care about, **[provided the stress
  load and test duration are meaningful]({{< relref "#stress-load"
  >}})**.

### Measuring response time to GPIO events {#latmus-gpio-response-time}

In addition to timer events, you will likely need to get a sense of
the worst case response time to common device interrupts you may
expect from EVL, as perceived by an application thread running in
user-space. This test mode is also available from the [latmus]({{<
relref "core/testing.md#latmus-program" >}}) program, when paired with
a [Zephyr-based](https://zephyproject.org) application which monitors
the response time from a remote system to the GPIO events it
sends.

{{% notice warning %}}
From the perspective of the monitor system, we will measure the
time it takes the SUT to not only receive the incoming event, but
also to respond to it by sending a converse GPIO acknowledge. Therefore
we expect the worst case figures to be higher than those reported by a
plain [response to timer test]({{< relref "#measuring-irq-response-time" >}}) .
{{% /notice %}}

For implementing this test, we need:

- a small development board which supports the [Zephyr
RTOS](https://zephyproject.org), and offers external GPIO pins. This
GPIO test was originally developed on a
[FRDM-K64F](https://www.nxp.com/design/development-boards/freedom-development-boards/mcu-boards/freedom-development-platform-for-kinetis-k64-k63-and-k24-mcus:FRDM-K64F)
board, a low-cost development platform from
[NXP](https://www.nxp.com/). For this reason, the Zephyr _device tree_
and _pinmux_ bits for enabling the GPIO lines are readily available
from [this
patch](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/frdm_k64f-enable-EVL-latency-monitor.patch).

- the latency monitoring application, aka
[latmon](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/),
which runs on the Zephyr board. This application periodically sends a
pulse on one GPIO (output) line to be received by the system under
test, then waits for an acknowledge on another GPIO (input) line,
measuring the time elapsed between the two events.

- the EVL-based system under test, also offering external GPIO
pins. This board runs [latmus]({{< relref
"core/testing.md#latmus-program" >}}) which sets up the test system,
asking
[latmon](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
to configure according to the settings requested by the user, then
enters a responder loop which listens to then acknowledges GPIO events
to the latency monitor using two separate GPIO lines. The period is
chosen by calling [latmus]({{< relref "core/testing.md#latmus-program"
>}}) using the `-p` option (which defaults to 1 ms, i.e. 1 Khz
sampling loop). This setting and a few others are passed by
[latmus]({{< relref "core/testing.md#latmus-program" >}}) to
[latmon](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
during the setup phase via the TCP/IP connection they share.

- a couple of wires between the GPIO pins of system under test and
  those of the Zephyr board running
  [latmon](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/),
  for transmitting pulses and receiving acknowledges to/from the
  system under test.

- a network connection between both systems so that [latmus]({{<
relref "core/testing.md#latmus-program" >}}) can establish a TCP/IP
stream with the remote
[latmon](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
application.

- a DHCP server reachable from the Zephyr board running on your
  LAN. The latency monitor asks for its IPv4 address by issuing a DHCP
  request at boot up.

The [latmon
application](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
running on the monitor board periodically raises a GPIO pulse event on
the TX wire, which causes the SUT to receive an interrupt. By calling
into the [EVL-enabled gpiolib]
(https://git.evlproject.org/linux-evl.git/tree/drivers/gpio/gpiolib.c?h=evl/master)
driver, the [responder thread]({{< relref "core/thread/_index.md" >}})
created by the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) waits for such interrupt by
monitoring edges on the GPIO pulse signal. Upon receipt, it
immediately acknowledges the event by raising an edge on the GPIO ack
signal, which
[latmon](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
monitors for calculating the latency value as the delay between the
acknowledge and pulse events. The latter accumulates these results,
sending an intermediate summary every second over a TCP/IP stream to a
logger thread running on the remote [latmus application]({{< relref
"core/testing.md#latmus-program" >}}).  This summary includes the
minimum, maximum and average latency values observed over the last 1Hz
period. Here again, the 1Hz display loop you can observe while
[latmus]({{< relref "core/testing.md#latmus-program" >}}) is running
is synchronized on the receipt of such summary.  The following figure
illustrates this execution flow:

![Alt text](/images/gpio-response-test.png "Response time to GPIO events")

#### Quick recipe: monitoring the [Raspberry PI 3](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) with the [FRDM-K64F](https://www.nxp.com/design/development-boards/freedom-development-boards/mcu-boards/freedom-development-platform-for-kinetis-k64-k63-and-k24-mcus:FRDM-K64F)

1. Build and install EVL on the Raspberry 3 as [described in this
document]({{< relref "core/build-steps.md" >}}).

2. Install the [Zephyr
SDK](https://docs.zephyrproject.org/latest/getting_started/installation_linux.html)
on your development system. Once the SDK is installed, if your are
using [Zephyr](https://zephyrproject.org) for the first time, you may
want to get your feet wet with the [blinky
example](https://docs.zephyrproject.org/latest/samples/basic/blinky/README.html). The
rest of the description assumes that the [Zephyr
SDK](https://docs.zephyrproject.org/latest/getting_started/installation_linux.html)
is rooted at ~/zephyrproject, and the [libevl source
tree](https://git.evlproject.org/libevl) was cloned into ~/libevl.

2. Patch the device tree and _pinmux_ changes to the FRDM-K64F board
which enable the GPIO lines in the Zephyr source tree:
```
$ cd ~/zephyrproject/zephyr
$ patch -p1 < ~/libevl/benchmarks/zephyr/frdm_k64f-enable-EVL-latency-monitor.patch
```

3. Connect the GPIO lines between both boards:

   - GPIO24 on the FRDM-K64F (pulse signal) should be wired to GPIO23
     on the RPI3 (ack signal, BCM numbering).

   - GPIO25 on the FRDM-K64F (ack signal) should be wired to GPIO24
     on the RPI3 (pulse signal, BCM numbering).

	{{% notice info %}}
   This GPIO wiring can also be used for testing the [Raspberry PI
   2](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/) and
   [Raspberry PI
   4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) the
   same way, since the GPIO pinout is identical.
   {{% /notice %}}

4. An [OpenSDA J-Link Onboard Debug
Probe](https://docs.zephyrproject.org/1.14.0/guides/debugging/probes.html#opensda-daplink-onboard-debug-probe)
is present on the FRDM-K64F, which can be used to flash the board. On
your development system, the [OpenOCD](https://docs.zephyrproject.org/1.14.0/guides/debugging/host-tools.html#openocd-debug-host-tools)
suite provides GDB remote debugging and flash programming support
compatible with this probe over USB. In most cases, a binary OpenOCD
package should be readily available from your favorite Linux
distribution. Once the OpenOCD suite is installed, you may need to add
some _udev_ rules in order for the USB device to appear on your
development system, such as [these
ones](https://github.com/zephyrproject-rtos/openocd/blob/master/contrib/60-openocd.rules).

5. Connect a USB cable from the _Open SDA_ micro-USB connector of the
FRDM-K64F to your development system. This will power on the FRDM-K64F
board, enabling firmware upload.

6. Build the [latmon
application](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
using the [Zephyr
SDK](https://docs.zephyrproject.org/latest/getting_started/installation_linux.html),
then flash it to the FRDM-K64F.
```
$ cd ~/zephyrproject/zephyr
$ source zephyr-env.sh
$ west build -p auto -b frdm_k64f ~/libevl/benchmarks/zephyr/latmon
-- west build: build configuration:
       source directory: ~/libevl/benchmarks/zephyr/latmon
       build directory: ~/zephyrproject/zephyr/build
       BOARD: frdm_k64f (origin: command line)
...
Memory region         Used Size  Region Size  %age Used
           FLASH:      100452 B         1 MB      9.58%
            SRAM:       45044 B       192 KB     22.91%
        IDT_LIST:         168 B         2 KB      8.20%
[156/160] Generating linker_pass_final.cmd
[157/160] Generating isr_tables.c
[158/160] Building C object zephyr/CMakeFiles/zephyr_final.dir/misc/empty_file.c.obj
[159/160] Building C object zephyr/CMakeFiles/zephyr_final.dir/isr_tables.c.obj
[160/160] Linking C executable zephyr/zephyr.elf
$ west flash
-- west flash: rebuilding
ninja: no work to do.
-- west flash: using runner pyocd
-- runners.pyocd: Flashing file: /home/rpm/git/zephyrproject/zephyr/build/zephyr/zephyr.bin
[====================] 100%
```

	Once (re-)flashed, the FRDM-K64F is automatically reset, with the
latency monitor taking over. You should see the following output in
the serial console of the FRDM-K64F:
```
*** Booting Zephyr OS build zephyr-<some version information> ***

[00:00:00.006,000] <inf> latency_monitor: DHCPv4 binding...
[00:00:03.001,000] <inf> eth_mcux: Enabled 100M full-duplex mode.
[00:00:03.003,000] <inf> net_dhcpv4: Received: <IP address of the FRDM-K64F>
[00:00:03.003,000] <inf> latency_monitor: DHCPv4 ok, listening on <IP>:2306
[00:00:03.003,000] <inf> latency_monitor: waiting for connection...
```

From that point, the latency monitor running on the FRDM-K64F is ready
to accept incoming connections from the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) running on the Raspberry PI 3 (SUT).

#### Native preemption and IRQ threading

The GPIO response time of the standard or
[PREEMPT_RT](https://wiki.linuxfoundation.org/realtime/rtl/blog)
kernel may be delayed by [threading the GPIO
interrupt](https://www.kernel.org/doc/htmldocs/kernel-api/API-request-threaded-irq.html)
the [latmus application]({{< relref "core/testing.md#latmus-program"
>}}) monitors, since this behavior is built in the generic GPIOLIB
driver. The overhead involved in waiting for a context switch to be
performed to the threaded handler increases the latency under stress
load. Disabling IRQ threading entirely in a single kernel
configuration (i.e. without EVL) would be the wrong option though,
making the latency figures generally really bad. However, you can
raise the priority of the IRQ thread serving the latency pulse above
any activity which should not be in its way, so that it is not delayed
even further.

In order to do this, you first need to locate the IRQ thread which
handles the GPIO pulse interrupts. A simple way to achieve this is to
check the output of `/proc/interrupts` once the test runs, looking for
the GPIO consumer called _latmon-pulse_. For instance, the following
output was obtained from a PREEMPT_RT kernel running on an i.MX8M SoM:

```
~ # cat /proc/interrupts 
           CPU0       CPU1       CPU2       CPU3       
  3:     237357      90193     227077     224949     GICv3  30 Level     arch_timer
  6:          0          0          0          0     GICv3  79 Level     timer@306a0000
  7:          0          0          0          0     GICv3  23 Level     arm-pmu
  8:          0          0          0          0     GICv3 128 Level     sai
  9:          0          0          0          0     GICv3  82 Level     sai
 20:          0          0          0          0     GICv3 110 Level     30280000.watchdog
 21:          0          0          0          0     GICv3 135 Level     sdma
 22:          0          0          0          0     GICv3  66 Level     sdma
...
 52:    1355491          0          0          0  gpio-mxc  12 Edge      latmon-pulse   <<< The one we look for
 55:          0          0          0          0  gpio-mxc  15 Edge      ds1337
 80:          0          0          0          0  gpio-mxc   8 Edge      bd718xx-irq
 84:          0          0          0          0  gpio-mxc  12 Edge      30b50000.mmc cd
...
```

With this information, we can now figure out which IRQ thread is
handling the pulse events monitored by [latmus]({{< relref
"core/testing.md#latmus-program" >}}), raising its priority as
needed. By default, all IRQ threads are normally set to priority 50 in
the SCHED_FIFO class. Typically, you may want to raise the priority of
this particular IRQ handler so that it does not have to compete with
other handlers. For instance, continuing the previous example we would
raise the priority of the kernel thread handling IRQ52 to 90:

{{% notice tip %}}
You can refine even futher the runtime configuration of a kernel threading
its interrupts by locking the [SMP
affinity](https://www.kernel.org/doc/Documentation/IRQ-affinity.txt)
of the IRQ threads on particular CPUs, in order to either optimize the wake
up time of processes waiting for such events, or reduce the jitter in
processing the real-time workload. Whether you should move
an IRQ thread to the isolated CPU also running the real-time workload
in order to favour locality, or keeping them spatially separate in
order to reduce the disturbance of interrupt handling on the workload
is something you may have to determine on a case-by-case basis.
{{% /notice %}}

```
~ # pgrep irq/52
345
~ # chrt -f -p 90 345
pid 345's current scheduling policy: SCHED_FIFO
pid 345's current scheduling priority: 50
pid 345's new scheduling policy: SCHED_FIFO
pid 345's new scheduling priority: 90
```

Which can be shortened as:

```
~ # chrt -f -p 90 $(pgrep irq/52)
pid 345's current scheduling policy: SCHED_FIFO
pid 345's current scheduling priority: 50
pid 345's new scheduling policy: SCHED_FIFO
pid 345's new scheduling priority: 90
```

{{% notice warning %}}
In a single kernel configuration, the IRQ thread process varies
between runs of the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) for the GPIO test, because the
interrupt descriptor is released at the end of each execution by
GPIOLIB. Make sure to apply the procedure explained above each time
you spawn a new test.
{{% /notice %}}

#### Running the GPIO-based test

Once the Zephyr board is started with the [latmon
application](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
flashed in, we can run the benchmark tests on the system under test.

This is done by running the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) on the SUT, passing either of
the `-Z` or `-z` option switch to select the [execution stage]({{<
relref "dovetail/altsched/_index.md#altsched-theory" >}}), depending
on whether we look for out-of-band response time figures (i.e. using
EVL) or plain in-band response time figures (i.e. _without_ relying on
EVL's real-time capabilities) respectively. In the latter case, we
would not even need EVL to be present in the kernel of the SUT;
typically we would use a
[PREEMPT_RT](https://wiki.linuxfoundation.org/realtime/rtl/blog)
kernel instead.

Regardless of the execution stage they should run on, both tests are
configured the same way. On the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) command line, we need to specify
which GPIO chip and pin number should be used for receiving GPIO
events (**-I \<gpiochip-name\>,\<pin-number\>**) and sending
acknowledge signals (**-O \<gpiochip-name\>,\<pin-number\>**).

When the test is started on the SUT, the [latmon
application](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
should start monitoring the GPIO response times of the [latmus
application]({{< relref "core/testing.md#latmus-program" >}})
indefinitely until the latter stops, at which point the latency
monitor goes back waiting for another connection.

```
[00:04:15.651,000] <inf> latency_monitor: monitoring started
/* ...remote latmus runs for some time... */
[00:04:24.877,000] <inf> latency_monitor: monitoring stopped
[00:04:24.879,000] <inf> latency_monitor: waiting for connection...
```

> Measuring out-of-band response time to GPIO events (on the SUT)
```
/*
 * Caution: the following output was produced by running the test only
 * a few seconds on an idle EVL-enabled system: the results displayed do not
 * reflect the worst case latency (which is higher) on this platform.
 */
# latmus -Z zephyr -I gpiochip0,23 -O gpiochip0,24
warming up on CPU1...
connecting to latmon at 192.168.3.60:2306...
RTT|  00:00:02  (oob-gpio, 1000 us period, priority 90, CPU1)
RTH|----lat min|----lat avg|----lat max|-overrun|---msw|---lat best|--lat worst
RTD|     11.541|     15.627|     18.125|       0|     0|     11.541|     18.125
RTD|     10.916|     15.617|     30.950|       0|     0|     10.916|     30.950
RTD|     12.500|     15.598|     25.908|       0|     0|     10.916|     30.950
RTD|      1.791|     15.571|     25.908|       0|     0|      1.791|     30.950
RTD|     13.958|     15.647|     26.075|       0|     0|      1.791|     30.950
^C
---|-----------|-----------|-----------|--------|------|-------------------------
RTS|      1.791|     15.606|     30.950|       0|     0|    00:00:05/00:00:05
```
> Measuring in-band response time to GPIO events (on the SUT)
```
/*
 * Caution: the following output was produced by running the test only
 * a few seconds on an idle PREEMPT_RT-enabled system: the results displayed do
 * not reflect the worst case latency (which is higher) on this platform.
 */
# latmus -z zephyr -I gpiochip0,23 -O gpiochip0,24 -P 99
warming up on CPU1...
connecting to latmon at 192.168.3.60:2306...
CAUTION: measuring in-band response time (no EVL there)
RTT|  00:00:02  (inband-gpio, 1000 us period, priority 99, CPU1)
RTH|----lat min|----lat avg|----lat max|-overrun|---msw|---lat best|--lat worst
RTD|     38.075|     52.401|     93.733|       0|     0|     38.075|     93.733
RTD|     39.700|     53.289|     91.608|       0|     0|     38.075|     93.733
RTD|     41.283|     54.914|     93.900|       0|     0|     38.075|     93.900
RTD|     38.075|     54.615|     91.608|       0|     0|     38.075|     93.900
RTD|     38.075|     54.767|     96.108|       0|     0|     38.075|     96.108
RTD|     41.283|     54.563|     91.608|       0|     0|     38.075|     96.108
^C
---|-----------|-----------|-----------|--------|------|-------------------------
RTS|     38.075|     54.037|     96.108|       0|     0|    00:00:07/00:00:07
```

### Configuring the test kernel for benchmarking {#test-kernel-config}

#### Hints for configuring any test kernel

- turn off all debug features and tracers in the kernel configuration.

- ensure all CPUs keep running at maximum frequency by enabling
  the "performance" CPU_FREQ governor, or disabling CPU_FREQ entirely.
 
- have a look at the [caveats here]({{< relref "core/caveat.md" >}}).

- make sure the GPU driver does not cause ugly latency peaks.

- isolate a CPU for running the latency test. For instance, you could
  reserve CPU1 for this purpose, by passing **isolcpus=1** on the
  kernel command line at boot.

#### Specific hints to configure a native preemption kernel

- enable maximum preemption (CONFIG_PREEMPT_RT_FULL if available).

- check that no common thread can compete with the responder thread on
  the same priority level. Some kernel threads will, but regular
  threads should not.
 
- switch to a non-serial terminal (ssh, telnet). Although this problem
  is being worked on upstream, significant output to a serial device
  might affect the worst case latency on some platforms with native
  preemption because of the implementation issues in console drivers,
  so the console should be kept quiet. You could also add the "quiet"
  option to the kernel boot arguments as an additional precaution.

#### Specific hints to configure an EVL-enabled kernel

- turn on CONFIG_EVL in the configuration.

- turn off all features from the CONFIG_EVL_DEBUG section. The cost of
  leaving the watchdog enabled should be marginal on the latency
  figures though.

### The issue of proper stress load {#stress-load}

Finding the worst case latency in measuring the response time to
interrupts requires applying a significant stress load to the system
in parallel to running the test itself. There have been many
discussions about what _significant_ should mean in this context. Some
have argued that real-time applications should have _reasonable_
requirements, defined by a set of restriction on their behavior and
environment, so that bounded response time can be guaranteed, which
sounds like asking application developers to abide by the rules
defined by kernel folks. When was the last time any of them did
so anyway?

Obviously, we cannot ask the infrastructure to be resilient to any
type of issue, including broken hardware or fatal kernel
bugs. Likewise, it is ok to define and restrict which API should be
used by applications to meet their real-time requirements. For
instance, there would be no point in expecting low and bounded latency
from all flavours of
[clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html) calls,
or whenever talking to some device involves a slow bus interface like
[i2c](https://i2c.info/). Likewise, we may impose some restrictions on
the kernel when it deals with these applications, like disabling
ondemand loading and copy-on-write mechanisms with
[mlock(2)](http://man7.org/linux/man-pages/man2/mlock.2.html).

---

Per [Murphy's
laws](http://www.murphys-laws.com/murphy/murphy-computer.html), we do
know that there is no point in wishful thinking, like hoping for
issues to never happen provided that we always do _reasonable_ things
which would meet some hypothetical standard of the
industry. Application developers do not always do _reasonable_ things,
they just do what they think is best doing, which almost invariably
differs from what kernel folks want or initially envisioned. After
all, the whole point of using Linux in this field is the ability to
combine real-time processing with the extremely rich GPOS feature set
such system provides, so there is no shortage of options and
varieties.  Therefore, when it comes to testing a real-time
infrastructure, let's _tickle the dragon's tail_.

---

There are many ways to stress a system, often depending on which kind
of issues we would like to trigger, and what follows does not pretend
to exhaustiveness. This said, these few aspects have proved to be
relevant over time when it comes to observing the worst case latency:

- Each time the kernel needs to switch between tasks which belong to
  distinct user address spaces, some MMU operations have to be
  performed in order to change the active memory context, which might
  also include costly cache maintenance in some cases. Those
  operations tend to take longer when the cache and memory subsystems
  have been under pressure at the time of the switch. Because the
  context switching process runs with interrupts disabled in the CPU,
  the higher the task switch rate, the more likely such extended
  interrupt masking may delay the response time to an external event.
 
- How the response time of the real-time infrastructure is affected by
  unfavourable cache situations is important. While no real-time work
  is pending, the real-time infrastructure just sleeps until the next
  event to be processed arrives. In the meantime, GPOS (as in
  non-realtime) activities may jump in, mobilizing all the available
  hardware resources for carrying out their work. As they do this,
  possibly treading on a lot of code and manipulating large volumes of
  data, the real-time program is gradually evicted from the CPU
  caches. When it resumes eventually in order to process an incoming
  event, it faces many cache misses, which induce delays. For this
  reason, and maybe counter-intuitively at first, the faster the timed
  loop the responder thread undergoes, the fewer the opportunities for
  the GPOS work to disturb the environment, the better the latency
  figures (up to a certain rate of course). On the contrary, a slower
  loop increases the likeliness of cache evictions when the kernel
  runs GPOS tasks while the real-time system is sleeping, waiting for
  the next event. If the CPU caches have been disturbed enough by the
  GPOS activities from the standpoint of the real-time work, then you
  may get closer to the actual worst case latency figures.

  In this respect, the - apparently - dull
  [dd(1)](http://man7.org/linux/man-pages/man1/dd.1.html) utility may
  become your worst nightmare as a real-time developer if you actually
  plan to assess the worst-case latency with your system. For
  instance, you may want to run this stupid workload in parallel to
  your favourite latency benchmark (hint: CPU isolation for the
  real-time workload won't save the day on most platforms):

  ```
  $ dd if=/dev/zero of=/dev/null bs=128M &
  ```

  {{% notice warning %}}
  Using a block factor of 128M is to make sure the loop will be
  disturbing the CPU caches enough. Too small a value here would only
  create a mild load, barely noticeable in the latency figures
  on many platforms.
  {{% /notice %}}

- A real-time application system is unlikely to be only composed of a
  single time-critical responder thread. We may have more real-time
  threads involved, likely at a lower priority though. So we need to
  assess the ability of the real-time infrastructure to schedule all
  of these threads efficiently. In this case, we want the responder
  thread to compete with other real-time threads for traversing the
  scheduler core across multiple CPUs in parallel. Efficient
  serialization of these threads within a CPU and between CPUs is
  key.

- Since we have to follow a probabilistic approach for determining the
  worst case latency, we ought to run the test long enough in order to
  increase the likeliness of exercizing the code path(s) which might
  cause the worst jitter. Practically, running the test under load for
  24 hours uninterrupted seems to deliver a worst case value we can
  trust.

#### Defining the stress workloads

Using Linux for running real-time workloads means that we have to meet
contradictory requirements on a shared hardware, maximum throughtput
and guaranteed response time at the same time, which on the face of
it, looks pretty insane, therefore interesting. Whichever real-time
infrastructure we consider, we have to assess how badly non real-time
applications might hurt the performances of real-time ones. With this
information, we can decide which is best for supporting a particular
real-time application on a particular hardware. To this end, the
following stress workloads are applied when running benchmarks:

1. As its name suggests, the _Mark Time_ workload is no workload at
   all. Under such conditions, the response time of the real-time
   system to some event (i.e. timer or GPIO interrupt) is measured
   while the GPOS is doing nothing in particular except waiting for
   something to do. The purpose is to get a sense of the best case we
   might achieve with a particular hardware and software
   configuration, unimpeded by GPOS activities.

2. The _Scary Grinder_ workload combines
   [hackbench](https://git.kernel.org/pub/scm/linux/kernel/git/clrkwllms/rt-tests.git)
   loops to a continuous
   [dd(1)](http://man7.org/linux/man-pages/man1/dd.1.html) copy from
   `/dev/zero` to `/dev/null` with a large block size (128Mb). This
   workload usually causes the worst latency spots on any platform for
   any type of real-time infrastructure, dual kernel and native
   preemption (PREEMPT_RT). As it pounds the CPU caches quite badly,
   it reveals the inertia of the real-time infrastructure when it has
   to ramp up quickly from an idle state in order to handle an
   external event. The shell commands to start this work are:

   ```
   ~ # while :; do hackbench; done &
   ~ # dd if=/dev/zero of=/dev/null bs=128M&
   ```

3. The _Pesky Neighbour_ workload is based on the [stress-ng test
   suite](https://github.com/ColinIanKing/stress-ng.git). Several
   "stressors" imposing specific loads on various kernel subsystems
   are run sequentially. The goal is to assess how sensitive the
   real-time infrastructure is to the pressure non real-time
   applications might put on the system by using common kernel
   interfaces. Given the ability some `stress-ng` stressors have to
   thrash and break the system, we limit the amount of damage they can
   do with specific settings, so that the machine stays responsive
   throughout long-running tests. Those settings depend on the compute
   power of the test machine. On a Raspberry PI 4B, this workload is
   started as follows:

   ```
   ~ # stress-ng --class cpu-cache --class scheduler \
       --sched other --taskset 0,2-$(nproc) \
       --sequential 1 --clone 1 --clone-max 200 --vm 3 --vm-bytes=64M \
       --exclude vforkmany --oomable
   ```

   There are a few points to notice:

   - `stress-ng` defines classes of stressors as shorthands one can
     use to refer to all of them implicitly. We focus on stressors
     which affect the scheduler and memory subsystems.

   - On a multi-core system, we always reserve CPU1 for running the
     test code which is monitored for response time, since native
     preemption requires some CPU(s) to be dedicated to the real-time
     workload for best results. This implies that **isolcpus=1
     evl.oobcpus=1** are passed to the kernel as boot options. For
     `stress-ng`, we also have to restrict the CPUs usable for running
     the stress load using the `--taskset` option. Assuming CPU1 is
     the only CPU isolated, this shell expression should produce the
     correct CPU set for that option: `0,2-$(nproc)`.

   - Some limits should be put on what `stress-ng` is allowed to do,
     in order to avoid bringing the machine down to a complete stall,
     or triggering OOM situations which may have unwanted outcomes
     such as wrecking the test entirely. Of course, those limits
     depend on the compute power of your test hardware.  With tiny
     embedded SoCs such as those from the
     [Raspberry](https://raspberrypi.org) family, you may have to cap
     the number of worker threads the `clone` stressor can run
     concurrently, and the number of clones such worker can create in
     turn. Likewise, the number of worker threads and the amount of
     virtual memory each of them maps should be kept within bounds for
     the `vm` stressor.

   - All threads running some stress workload should belong to the
     [SCHED_OTHER](http://man7.org/linux/man-pages/man7/sched.7.html)
     scheduling class. They should NOT compete with the real-time
     threads whose response time is being monitored during the
     test. Although this would have actually no impact on the
     real-time performances of a dual kernel system, this would lead
     to an unfair comparison with a native preemption system such as
     PREEMPT_RT which is sensitive to this issue by design.

### The benchmarks we do

![Alt text](/images/wip.png "To be continued")

---

{{<lastmodified>}}
