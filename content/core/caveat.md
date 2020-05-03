---
menuTitle: "Caveat"
weight: 7
---

# Things you definitely want to know

## Generic issues

### **isolcpus** is our friend too {#caveat-isocpus}

Isolating some CPUs on the kernel command line using the _isolcpus=_
option, in order to prevent the load balancer from offloading in-band
work to them is not only a good idea with
[PREEMPT_RT](https://wiki.linuxfoundation.org/realtime/rtl/blog), but
for any dual kernel configuration too.

By doing so, having some random in-band work evicting cache lines on a
CPU where real-time threads briefly sleep is less likely, increasing
the odds of costly cache misses, which translates positively into the
latency numbers you can get. Even if EVL's small footprint core has a
limited exposure to such kind of disturbance, saving a handful of
microseconds is worth it when the worst case figure is already within
tenths of microseconds.

### **CONFIG_DEBUG_HARD_LOCKS** is cool but ruins real-time guarantees

When CONFIG_DEBUG_HARD_LOCKS is enabled, the lock dependency engine
(CONFIG_LOCKDEP) which helps in tracking down deadlocks and other
locking-related issues is also enabled for Dovetail's [hard
locks]({{% relref "dovetail/pipeline/locking.md#new-spinlocks" %}}),
which underpins most of the serialization mechanisms the EVL core
uses.

This is nice as it has the lock validator monitor the hard spinlocks
EVL uses too. However, this comes with a high price latency-wise:
seeing hundreds of microseconds spent in the validator with hard
interrupts off from time to time is not uncommon. Running the latency
monitoring utility (aka `latmus`) which is part of `libevl` in this
configuration should give you pretty ugly numbers.

In short, it is fine enabling CONFIG_DEBUG_HARD_LOCKS for debugging
some locking pattern in EVL, but you won't be able to meet real-time
requirements at the same time in such configuration.

### CPU frequency scaling (usually) has a negative impact on latency {#caveat-cpufreq}

Enabling the _ondemand_ CPUFreq governor - or any governor performing
dynamic adjustment of the CPU frequency - may induce significant
latency for EVL on your system, from ten microseconds to more than a
hundred depending on the hardware. Selecting the so-called
_performance_ governor is the safe option, which guarantees that no
frequency transition ever happens, keeping the CPUs at their maximum
processing speed.

In other words, if CONFIG_CPU_FREQ has to be enabled in your
configuration, enabling CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE and
CONFIG_CPU_FREQ_GOV_PERFORMANCE exclusively is most often the best way
to prevent unexpectedly high latency peaks.

### Disable CONFIG_SMP for best latency on single-core systems

On single-core hardware, some out-of-line code may still be executed
for dealing with various types of spinlock with a SMP build, which
translates into additional CPU branches and cache misses. On low end
hardware, this overhead may be noticeable.

Therefore, if you neither need SMP support nor kernel debug options
which depend on instrumenting the spinlock constructs (e.g.
CONFIG_DEBUG_PREEMPT), you may want to disable all the related kernel
options, starting with CONFIG_SMP.

## Architecture-specific issues

### x86 {#x86-caveat}

- GCC 10.x might generate code causing the SMP boot process to break
  early, as reported [by this
  post](https://lkml.org/lkml/2020/3/14/186). As a work-around, you
  can disable `CONFIG_STACKPROTECTOR_STRONG` from your kernel
  configuration.

- CONFIG_ACPI_PROCESSOR_IDLE may increase the latency upon wakeup on
  IRQ from idle on some SoC (up to 30 us observed) on x86. This option
  is implicitly selected by the following configuration chain:
  CONFIG_SCHED_MC_PRIO &#8594; CONFIG_INTEL_PSTATE &#8594;
  CONFIG_ACPI_PROCESSOR. If out-of-range latency figures are observed
  on your x86 hardware, turning off this chain may help.

- When the HPET is disabled, the watchdog which monitors the sanity of
  the current clocksource for the kernel may use _refined-jiffies_ as
  the reference clocksource to compare with. Unfortunately, such
  clocksource is fairly imprecise for timekeeping since timer
  interrupts might be missed.  This could in turn trigger false
  positives with the watchdog, which would end up declaring the TSC
  clocksource as 'unstable'. For instance, it has been observed that
  enabling CONFIG_FUNCTION_GRAPH_TRACER on some legacy hardware would
  systematically cause such behavior at boot. The following warning
  splat appearing in the kernel log is symptomatic of this problem:

  ```
  clocksource: timekeeping watchdog on CPU0: Marking clocksource
               'tsc-early' as unstable because the skew is too large:
  clocksource: 'refined-jiffies' wd_now: fffb7018 wd_last: fffb6e9d 
               mask: ffffffff
  clocksource: 'tsc-early' cs_now: 68a6a7070f6a0 cs_last: 68a69ab6f74d6 
               mask: ffffffffffffffff
  tsc: Marking TSC unstable due to clocksource watchdog
  ```

	This is a problem because the TSC is the best-rated
clocksource and [directly accessible from the vDSO]({{< relref
"dovetail/porting/clocksource.md#time-vdso-access" >}}), which speeds
up timestamping operations. If the TSC on your hardware is known to be
fine and face this issue nevertheless, you may want to pass
**tsc=nowatchdog** to the kernel to prevent it, or even
**tsc=reliable** if all TSCs are reliable enough to be synchronized
across CPUs.  If the TSC is really unstable on some legacy hardware
and you cannot ignore the watchdog alert, you can still leave it to
other clocksources such as _acpi\_pm_. Calls to [evl_read_clock()]({{<
relref "core/user-api/clock/_index.md#evl_read_clock" >}}) would be
significantly slower compared to a direct readout from the vDSO, but
the EVL core would manage to get timestamps from its [built-in
clocks]({{< relref "core/user-api/clock/_index.md#builtin-clocks" >}})
from the out-of-band stage at the expense of a system call, without
involving the in-band stage.

  	You can retrieve the current clocksource used by the kernel as follows:

```
# cat /sys/devices/system/clocksource/clocksource0/current_clocksource
tsc
```
 
- NMI-based _perf_ data collection may cause the kernel to execute
  utterly sluggish ACPI driver code at each event. Since disabling
  CONFIG_PERF is not an option, passing **nmi_watchodg=0** on the
  kernel command line at boot may help.

{{% notice warning %}}
Passing **nmi_watchodg=0** turns off the hard lockup detection for the
in-band kernel. However, by enabling CONFIG_EVL_WATCHDOG, EVL will
still detect runaway EVL threads stuck in out-of-band execution.
{{% /notice %}}

---

{{<lastmodified>}}
