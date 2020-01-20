---
menuTitle: "Benchmarking"
title: "Running benchmarks"
weight: 5
---

### Measuring the response time to interrupts {#measuring-irq-response-time}

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
summarizes the potential delays which may exist between the moment an
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

### Measuring the response time to timer events {#latmus-timer-response-time}

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
the receipt of such summary. The execution flow just described can be
summarized as follows:

![Alt text](/images/timer-response-test.png "Response time to timer events")

When the test completes, the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) determines the minimum, worst
case and average latency values over the whole test duration. Upon
request by passing the _-g_ option, the [latmus application]({{<
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
>}}) using the _-m_ option, which can be omitted since measuring the
response time to timer events is the default test.

> Measuring the response time to timer events
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

    The [calibration settings](({{< relref "core/timer-calibration.md"
    >}})) of the EVL core clock which applied during the test.

- _\# clocksource: tsc_

    The name of the [kernel clock source]({{< relref
    "dovetail/porting/clocksource.md" >}}) used by the EVL core for
    reading timestamps. This value depends on the processor
    architecture, _tsc_ commonly refers to x86.

- _\# vDSO access: architected_

    The type of access the application had to the [kernel clock
    source]({{< relref "dovetail/porting/clocksource.md" >}}) via the
    vDSO. The following values are defined:

  - _architected_ denotes a fast (syscall-less) vDSO access to a
    built-in clock source defined by the architecture itself. This is
    the best case.

  - _mmio_ denotes a fast (syscall-less) vDSO access to a [clock source]
    ({{< relref
    "dovetail/porting/clocksource.md#generic-clocksource-vdso" >}})
    exported via Dovetail's generic access to MMIO-based devices. This
    is the second best case.

  - _none_ denotes a not-so-fast access to a kernel clock source
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
      [SCHED_FIFO]({{< relref "core/user-api/scheduling#SCHED_FIFO"
      >}}) class. By definition, any legit value starting from 1 and
      on gives the responder thread higher priority than any in-band
      task running in the system, including kernel threads of any
      sort.

- _\# thread affinity: CPU1_

      The processor affinity of the responder thread. If the CPU
      id. is suffixed by **-noisol**, then the responder was not
      running on an isolated processor during the test, which most
      likely entailed higher latency values compared to isolating this
      CPU - unless you did not [stress]({{< relref "#stress-load" >}})
      the SUT enough or at all while measuring, which would make the
      figures obtained quite flawed and probably worthless anyway.

- _\# C-state restricted_

      Whether the processor C-State was restricted to the shallowest
      level, preventing it to enter deeper sleep states which are
      known to induce extra latency.

- _\# duration (hhmmss): 00:01:12_

      How long the test ran (according to the wall clock).

- _\# peak (hhmmss): 00:00:47_

      When the worst case value was observed, relatively to the
      beginning of the test.

- _\# min latency: 0.205_

      The best/shortest latency value observed throughout the test.

- _\# avg latency: 3.097_

      The average latency value observed, accounting for all
      measurement samples collected during the test.

- _\# max latency: 25.510_

      The maximum value observed throughout the test. This is the
      **worst case** value we should definitely care about,
      **[provided the stress load and test duration are
      meaningful]({{< relref "#stress-load" >}})**.

### Measuring the response time to GPIO events {#latmus-gpio-response-time}

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
>}}) using the _-p_ option (which defaults to 1 ms, i.e. 1 Khz
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
is synchronized on the receipt of such summary. The execution flow
just described can be summarized as follows:

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
your development system, the [OpenOCD]
(https://docs.zephyrproject.org/1.14.0/guides/debugging/host-tools.html#openocd-debug-host-tools)
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
interrupt](https://www.kernel.org/doc/htmldocs/kernel-api/API-request-threaded-irq.html),
since this behavior is built in the generic GPIOLIB driver. The
overhead involved in waiting for a context switch to be performed to
the threaded handler definitely shows badly in the figures, especially
under stress load.

Unfortunately, disabling IRQ threading entirely in a single kernel
configuration (i.e. without EVL) would be the wrong option though,
making the latency figures even worse. The following patch allows to
disable IRQ threading on demand, only for GPIO chip drivers which
cannot (more precisely: do not have to) sleep when accessing the chip
registers. Once patched in, the kernel can be passed the
**gpiolib.nothread** option at boot, which disables IRQ threading
whenever possible for handling GPIO interrupts.

**You definitely do not need this patch when running EVL, there is no
such IRQ threading in the EVL-enabled GPIOLIB driver.**

{{% notice warning %}}
The following code is a **hack**, originally written on top of
v5.4.5-rt3 only for the purpose of running comparable test cases
between EVL and native preemption systems when it comes to measuring
their response time to GPIO events. It is not intended to be a
canonical solution to this problem.
{{% /notice %}}

```
diff --git a/drivers/gpio/gpiolib.c b/drivers/gpio/gpiolib.c
index 104ed299d5ea..77f78fda96f1 100644
--- a/drivers/gpio/gpiolib.c
+++ b/drivers/gpio/gpiolib.c
@@ -35,6 +35,10 @@
 #define CREATE_TRACE_POINTS
 #include <trace/events/gpio.h>
 
+static bool nothread;
+module_param(nothread, bool, 0444);
+MODULE_PARM_DESC(nothread, "Do not thread GPIO interrupt events");
+
 /* Implementation infrastructure for GPIO interfaces.
  *
  * The GPIO programming interface allows for inlining speed-critical
@@ -866,6 +870,11 @@ static irqreturn_t lineevent_irq_thread(int irq, void *p)
 	return IRQ_HANDLED;
 }
 
+static inline bool lineevent_no_thread(struct lineevent_state *le)
+{
+	return nothread && !le->gdev->chip->can_sleep;
+}
+
 static irqreturn_t lineevent_irq_handler(int irq, void *p)
 {
 	struct lineevent_state *le = p;
@@ -875,6 +884,8 @@ static irqreturn_t lineevent_irq_handler(int irq, void *p)
 	 * close in time as possible to the actual event.
 	 */
 	le->timestamp = ktime_get_real_ns();
+	if (lineevent_no_thread(le))
+		return lineevent_irq_thread(irq, p);
 
 	return IRQ_WAKE_THREAD;
 }
@@ -971,7 +982,8 @@ static int lineevent_create(struct gpio_device *gdev, void __user *ip)
 	/* Request a thread to read the events */
 	ret = request_threaded_irq(le->irq,
 			lineevent_irq_handler,
-			lineevent_irq_thread,
+			lineevent_no_thread(le) ? NULL :
+				lineevent_irq_thread,
 			irqflags,
 			le->label,
 			le);
```

#### Running the GPIO-based test

Once the Zephyr board is started with the [latmon
application](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
flashed in, we can run the benchmark tests on the system under test.

This is done by running the [latmus application]({{< relref
"core/testing.md#latmus-program" >}}) on the SUT, passing either of
the _-Z_ or _-z_ option switch to select the [execution stage]({{<
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

> Measuring the out-of-band response time to GPIO events (on the SUT)
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
> Measuring the in-band response time to GPIO events (on the SUT)
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

- ensure that all CPUs keep running at maximum frequency by enabling
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
defined by kernel folks. **_When was the last time any of them did
so?_**

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

- each time the kernel needs to switch between tasks which belong to
  distinct user address spaces, some MMU operations have to be
  performed in order to change the active memory context, which might
  also include costly cache maintenance in some cases. Those
  operations tend to take longer when the cache and memory subsystems
  have been under pressure at the time of the switch. Because the
  context switching process runs with interrupts disabled in the CPU,
  the higher the task switch rate, the more likely such extended
  interrupt masking may delay the response time to an external event.
 
- how the response time of the real-time infrastructure is affected by
  unfavourable cache situations is important. While no real-time work
  is pending, the real-time infrastructure just sleeps until the next
  event to be processed arrives. In the meantime, GPOS (as in
  non-realtime) activities may jump in, mobilizing all the available
  hardware resources for carrying out their work. As they do this,
  possibly treading on a lot of code and manipulating large volumes of
  data, the real-time program is gradually evicted from the CPU
  caches. When it resumes eventually in order to process an incoming
  event, it faces many cache misses, which induces delays. For this
  reason, and maybe counter-intuitively at first, the faster the timed
  loop the responder thread undergoes, the fewer the opportunities for
  the GPOS work to disturb the environment, the better the latency
  figures (up to a certain rate of course). On the contrary, a slower
  loop increases the likeliness of cache evictions when the kernel
  runs GPOS tasks while the real-time system is sleeping, waiting for
  the next event. If the CPU caches have been disturbed enough by the
  GPOS activities from the standpoint of the real-time work, then you
  may get closer to the actual worst case latency figures.

- a real-time application system is unlikely to be only composed of a
  single time-critical responder thread. We may have more real-time
  threads involved, likely at a lower priority though. So we need to
  assess the ability of the real-time infrastructure to schedule all
  of these threads efficiently. In this case, we want the responder
  thread to compete with other real-time threads for traversing the
  scheduler core across multiple CPUs in parallel. Efficient
  serialization of these threads within a CPU and between CPUs is
  key.

- since we have to follow a probabilistic approach for determining the
  worst case latency, we ought to run the test long enough in order to
  increase the likeliness of exercizing the code path(s) which might
  cause the worst jitter. Practically, running the test under load for
  24 hours uninterrupted seems to deliver a worst case value we can
  trust.

#### Defining the stressing toolkit

In addition to the [latmus]({{< relref
"core/testing.md##latmus-program" >}}) and
[latmon](https://git.evlproject.org/libevl.git/tree/benchmarks/zephyr/latmon/)
utilities described ealier in this document, the following software
will be used in the test scenarios:

- The [dd(1)](http://man7.org/linux/man-pages/man1/dd.1.html) command.

- The _cyclictest_ and _hackbench_ programs as available from the
  [PREEMPT-RT test
  suite](https://git.kernel.org/pub/scm/linux/kernel/git/clrkwllms/rt-tests.git).

- The [stress-ng tests suite](https://github.com/ColinIanKing/stress-ng.git).


![Alt text](/images/wip.png "To be continued")
