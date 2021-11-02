---
menuTitle: "Предостережение"
title: "Вещи, которые вы определенно хотите знать"
weight: 7
original_path: "content/core/caveat.md"
original_hash: "129a74f"
---

## Общие вопросы

### `isolcpus` это тоже наш друг {#caveat-isocpus}

Изолировать некоторые процессоры в командной строке ядра с помощью опции
_isolcpus=_, чтобы не допустить, чтобы балансировщик нагрузки перегружал на них
внутриполосную работу, - это не только хорошая идея с
[PREEMPT_RT](https://wiki.linuxfoundation.org/realtime/rtl/blog), но и для
любого инструмента настройки двойного ядра.

Таким образом, некоторая случайная внутриполосная работа по удалению строк кэша
на процессоре, где потоки реального времени ненадолго отключаются, менее вероятна,
что увеличивает вероятность дорогостоящих пропусков кэша, что положительно
сказывается на значениях задержек, которые вы можете получить. Даже если
небольшое ядро EVL имеет ограниченную подверженность воздействию такого рода помех,
экономия нескольких микросекунд стоит того, когда наихудший показатель
уже находится в пределах десятых долей микросекунд.

### `CONFIG_DEBUG_HARD_LOCKS` это круто, но разрушает гарантии в реальном времени

Когда включена функция `CONFIG_DEBUG_HARD_LOCKS`, механизм зависимостей
блокировки (`CONFIG_LOCKDEP`), который помогает отслеживать взаимоблокировки и
другие проблемы, связанные с блокировками, также включен для
[жестких блокировок]({{% relref "dovetail/pipeline/locking#new-spinlocks" %}})
Ласточкиного хвоста (Dovetail), который лежит в основе большинства механизмов
сериализации, используемых ядром EVL.

Это хорошо, так как у него есть валидатор блокировки, который также контролирует
жесткие блокировки, используемые EVL. Однако это связано с высокой ценой с точки
зрения задержки: не редкость видеть сотни микросекунд, проведенных в валидаторе,
время от времени с жесткими прерываниями. Запуск утилиты мониторинга задержки
(она же `latmus`) которая является частью `libevl` в этой конфигурации, должен
дать вам довольно уродливые цифры.

Короче говоря, это нормально, если включить `CONFIG_DEBUG_HARD_LOCKS` для отладки
некоторого шаблона блокировки в EVL, но вы не сможете одновременно выполнять
требования в режиме реального времени в такой конфигурации.

### Масштабирование частоты процессора (обычно) оказывает негативное влияние на задержку {#caveat-cpufreq}

Enabling the _ondemand_ CPUFreq governor - or any governor performing
dynamic adjustment of the CPU frequency - may induce significant
latency for EVL on your system, from ten microseconds to more than a
hundred depending on the hardware. Selecting the so-called
_performance_ governor is the safe option, which guarantees that no
frequency transition ever happens, keeping the CPUs at their maximum
processing speed.

In other words, if `CONFIG_CPU_FREQ` has to be enabled in your
configuration, enabling `CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE` and
`CONFIG_CPU_FREQ_GOV_PERFORMANCE` exclusively is most often the best way
to prevent unexpectedly high latency peaks.

### Disable `CONFIG_SMP` for best latency on single-core systems

On single-core hardware, some out-of-line code may still be executed
for dealing with various types of spinlock with a SMP build, which
translates into additional CPU branches and cache misses. On low end
hardware, this overhead may be noticeable.

Therefore, if you neither need SMP support nor kernel debug options
which depend on instrumenting the spinlock constructs (e.g.
`CONFIG_DEBUG_PREEMPT`), you may want to disable all the related kernel
options, starting with `CONFIG_SMP`.

## Architecture-specific issues

### x86 {#x86-caveat}

- GCC 10.x might generate code causing the SMP boot process to break
  early, as reported [by this
  post](https://lkml.org/lkml/2020/3/14/186). As a work-around, you
  can disable `CONFIG_STACKPROTECTOR_STRONG` from your kernel
  configuration.

- `CONFIG_ACPI_PROCESSOR_IDLE` may increase the latency upon wakeup on
  IRQ from idle on some SoC (up to 30 us observed) on x86. This option
  is implicitly selected by the following configuration chain:
  `CONFIG_SCHED_MC_PRIO` &#8594; `CONFIG_INTEL_PSTATE` &#8594;
  `CONFIG_ACPI_PROCESSOR`. If out-of-range latency figures are observed
  on your x86 hardware, turning off this chain may help.

- When the HPET is disabled, the watchdog which monitors the sanity of
  the current clocksource for the kernel may use _refined-jiffies_ as
  the reference clocksource to compare with. Unfortunately, such
  clocksource is fairly imprecise for timekeeping since timer
  interrupts might be missed.  This could in turn trigger false
  positives with the watchdog, which would end up declaring the TSC
  clocksource as 'unstable'. For instance, it has been observed that
  enabling  `CONFIG_FUNCTION_GRAPH_TRACER` on some legacy hardware would
  systematically cause such behavior at boot. The following warning
  splat appearing in the kernel log is symptomatic of this problem:

  ```log
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
"dovetail/porting/clocksource#time-vdso-access" >}}), which speeds
up timestamping operations. If the TSC on your hardware is known to be
fine and face this issue nevertheless, you may want to pass
`tsc=nowatchdog` to the kernel to prevent it, or even `tsc=reliable`
if all TSCs are reliable enough to be synchronized across CPUs.  If
the TSC is really unstable on some legacy hardware and you cannot
ignore the watchdog alert, you can still leave it to other
clocksources such as _acpi\_pm_. Calls to [evl_read_clock()]({{<
relref "core/user-api/clock/_index.md#evl_read_clock" >}}) would be
slower compared to a direct syscall-less readout from the vDSO, but
the EVL core would nevertheless manage to get timestamps from its
[built-in clocks]({{< relref
"core/user-api/clock/_index.md#builtin-clocks" >}}) at the expense of
an out-of-band system call, without involving the in-band stage
though. You definitely want to make sure everything is right on your
platform with respect to reading timestamps by running the
[latmus]({{< relref "core/testing#latmus-program" >}}) test, which
can detect any related issue.

  	You can retrieve the current clocksource used by the kernel as follows:

```sh
# cat /sys/devices/system/clocksource/clocksource0/current_clocksource
tsc
```
 
- NMI-based _perf_ data collection may cause the kernel to execute
  utterly sluggish ACPI driver code at each event. Since disabling
  `CONFIG_PERF` is not an option, passing `nmi_watchodg=0` on the
  kernel command line at boot may help.

{{% notice warning %}}
Passing `nmi_watchodg=0` turns off the hard lockup detection for the
in-band kernel. However, EVL will still detect runaway EVL threads
stuck in out-of-band execution if `CONFIG_EVL_WATCHDOG` is enabled.
{{% /notice %}}

---

{{<lastmodified>}}
