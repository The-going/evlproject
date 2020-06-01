---
menuTitle: "Clock sources"
title: "Reading clock sources"
weight: 95
---

Your autonomous core most likely needs a fast access to the current
clock source from the out-of-band context, for reading precise
timestamps which are in sync with the kernel's idea of time. The best
way to achieve this is by enabling the fast
[clock_gettime(3)](http://man7.org/linux/man-pages/man3/clock_gettime.3.html)
helper in the [vDSO support](https://lwn.net/Articles/615809/) for the
target CPU architecture. At least, you may want user-space tasks
controlled by the core to have access to the POSIX-defined
`CLOCK_MONOTONIC` and `CLOCK_REALTIME` clocks from the out-of-band
context, using a vDSO call, with no execution and response time
penalty involved in invoking an [in-band syscall] ({{< relref
"dovetail/altsched#inband-switch" >}}).

### Reading timestamps via the vDSO in a nutshell {#time-vdso-access}

Basically, A vDSO-based
[clock_gettime(3)](http://man7.org/linux/man-pages/man3/clock_gettime.3.html)
implementation wants to read from non-privileged CPU mode the same
monotonic hardware counter which is currently used by the kernel for
timekeeping, before converting the count to nanoseconds. In other
words, this implementation should mirror the kernel
[clocksource](https://www.kernel.org/doc/Documentation/timers/timekeeping.txt)
behavior. However, it should do so relying exclusively on resources
which may be accessed from user mode, since the vDSO code segment
containing this helper is merely a kernel-defined extension of the
application code running in non-privileged mode. In order to perform
such conversion, the vDSO code also needs additional timekeeping
information maintained by the kernel, which it usually gets from a
small data segment the kernel maps into every application as part of
the vDSO support (see the various _update\_vsyscall()_ implementations
for more on this).

There are two common ways of reading the hardware counter used for
timekeeping by the kernel from a non-privileged environement:

- either by reading some non-privileged on-chip register
  (e.g. _powerpc_'s timebase register, or _x86_'s TSC register).

- or by reading a memory-mapped counter, from a memory mapping the
  calling application has access to.

### The ARM _situation_

For several CPU architectures Linux supports, reading the
CLOCK_MONOTONIC and CLOCK_REALTIME clocks is already possible via vDSO
calls, including from tasks [running out-of-band]({{< relref
"dovetail/altsched#altsched-theory" >}}) without incurring
any [execution stage switch]({{< relref
"dovetail/altsched#stage-migration" >}}).

For ARM, the situation is clumsy: the mainline kernel implementation
supports reading timestamps directly from the vDSO **only** for the
so-called _architected timer_ the armv8 specification requires from
compliant CPUs. With CPUs following an earlier specification, a
truckload of different hardware chips may be used for that purpose
instead, which the vDSO implementation does not provide any support
for. Sometimes, the architected timer is present but not usable for
timekeeping duties because of firmware issues. In these cases, a plain
in-band system call would be issued whenever the vDSO-based
[clock_gettime(3)](http://man7.org/linux/man-pages/man3/clock_gettime.3.html)
is called from an application, which would be a showstopper for
keeping the response time short and bounded.

### The generic vDSO and USER_MMIO clock sources {#generic-clocksource-vdso}

If your target kernel supports the generic vDSO implementation
(CONFIG_HAVE_GENERIC_VDSO), Dovetail already provides the core support
for accessing MMIO-based clock sources from the vDSO, which is
essentially part of the application context:

- firstly, it extends the _clocksource\_mmio_ semantics with the
MMIO-accessed clock sources mapped to user space aka _struct
clocksource\_user\_mmio_, implemented in `drivers/clocksource/mmio.c`,
which we call USER_MMIO for short in this document.  In essence, a
USER_MMIO clock source is a MMIO-based clock source which any
application may map into its own address space, so that it can read
the hardware counter directly. Specifically, the MMIO space covering
the hardware counter register(s) is mapped into the caller's address
space.

- Secondly, Dovetail extends the generic vDSO implementation in
`lib/vdso.c` and `kernel/time/vsyscall.c` in order to make USER_MMIO
clock sources visible to applications, in addition to the architected
timer if present.

- Finally, Dovetail converts some MMIO clock sources to USER_MMIO
clock sources, such as the OMAP2 general-purpose timer, the ARM global
timer counter and the DesignWare APB timer. More conversions will come
over time, as Dovetail is ported to a broader range of hardware.

{{% notice note %}}
The mapping operation happens once in a process lifetime, during the
very first call to [clock_gettime(3)](http://man7.org/linux/man-pages/man3/clock_gettime.3.html)
issued by the application. Since this involves running in-band code for
updating the caller's address space, this particular call gives absolutely
**no** response time guarantee. So it's good practice to force an initial
dummy call to [clock_gettime(3)](http://man7.org/linux/man-pages/man3/clock_gettime.3.html)
from the library code which initializes the
interface between applications and your autonomous core (i.e. some
user-space library which implements the out-of-band system calls
wrappers to send requests to this core). For EVL, this is done in
[libevl]({{< relref "core/user-api/_index.md" >}}).
{{% /notice %}}

#### Making a MMIO clock source accessible from the vDSO {#enabling-mmio-clocksource}

If you need to convert an existing MMIO clock source to a
user-mappable one visible from the generic vDSO, you can follow this
three-step process:

- in the Kconfig stanza enabling the clock source, select
  `[CONFIG_]GENERIC_CLOCKSOURCE_VDSO` in the dependency list to
  compile the required support in the generic vDSO code, which in turn
  selects the USER_MMIO support it depends on.

- in the clock source implementation, convert the _struct clocksource_
  descriptor to a _struct clocksource\_user\_mmio_ descriptor. The
  original _struct clocksource_ object is now available as a member of
  the _struct clocksource\_user\_mmio_ descriptor, so you may have to
  move the original initializers accordingly. You also need to fix up
  the clock source's `.read()` handler, changing it to one of the
  helpers _clocksource\_user\_mmio_ knows about. Do _not_ use any
  other helper outside of the following set, or you would receive
  -EINVAL from `clocksource_user_mmio_init()`:

  * `clocksource_mmio_readl_up()`, for reading a 32bit count-up
    register (i.e. _reg\_higher_ is NULL in _clocksource\_mmio\_regs_
    as described below).

  * `clocksource_mmio_readl_down()`, for reading a 32bit count-down
    register.

  * `clocksource_mmio_readw_up()`, for reading a 16bit count-up
    register.

  * `clocksource_mmio_readw_down()`, for reading a 16bit count-down
    register.

  * `clocksource_dual_mmio_readl_up()`, for reading a count-up counter
    composed of two 32bit registers (i.e. both _reg\_lower_ and
    _reg\_higher_ must be valid in _clocksource\_mmio\_regs_ as
    described below).

  * `clocksource_dual_mmio_readl_down()`, for reading a count-down
    counter composed of two 32bit registers.
    
{{% notice warning %}}
Only continuous clock sources can be converted to
_clocksource\_user\_mmio_, otherwise the registration fails with
-EINVAL in `clocksource_user_mmio_init()`. Therefore, only clock
sources originally bearing the `CLOCK_SOURCE_IS_CONTINUOUS` flag can
be converted.
{{% /notice %}}

- eventually, substitute the call to `clocksource_register_hz()` by a
  call to `clocksource_user_mmio_init()` instead. This function takes
  the following arguments:

  * the USER_MMIO descriptor address

  * the address of a _clocksource\_mmio\_regs_ structure which defines
    the method and parameters for reading the hardware counter.  Such
    counter can be represented by up to two 32bit MMIO registers,
    making a 64bit value. A lower precision is acceptable too, the
    vDSO code deals with wrapping as needed. However, the higher the
    precision, the better the accuracy for applications. The
    _clocksource\_mmio\_regs_ structure should be filled with the
    following information:

         - _reg\_lower_ is the **virtual** address of the counter's
      	   low 32bit register. This address was most likely obtained
      	   from `ioremap()` in the original clock source driver code;
      	   it cannot be NULL.

	 - _bits\_lower_ is a bitmask defining the significant bits to
	   read from the low register, starting from the low order
	   bit. For instance, if the first 31 bits only are
	   significant, 0x7fffffff should be passed.

	 - _reg\_higher_ is the **virtual** address of the counter's
	   high register. This address can be NULL if the hardware
	   counter is only 32bit wide or less, in which case
	   _bits\_higher_ is ignored too.

	 - _bits\_higher_ is a bitmask defining the significant bits to read
           from the high register.

	 - _revmap_ is a reverse mapping helper, for resolving the
           physical address of the low and high registers mentioned
           above, based on the virtual address passed in _reg\_lower_
           and _reg\_higher_. If _revmap_ is NULL,
           `clocksource_user_mmio_init()` tries to figure this out by
           resolving the address of the containing memory frame via a
           call to `find_vma()`, which is usually fine. If this
           resolution should be done in a different way, you should
           specify your own handler in _revmap_, which receives the
           **virtual** address to resolve, and should return the
           corresponding physical address, or zero upon failure.
   
  * the hardware clock rate that was originally passed to
    `clocksource_register_hz()`.

### Example: converting the OMAP2 GP-timer

The Beaglebone Black is an AM335X processor equipped with a Cortex A8
CPU, therefore no ARM architected timer is available. Instead, Linux
runs one of the available general purpose timers on this platform for
timekeeping purpose. The clock source driver for such devices is
implemented in `arch/arm/mach-omap2/timer.c`. The patch below
illustrates the changes Dovetail introduces to convert this clock
source to a user-mappable one, which the ARM vDSO implementation can
use.

```
--- a/arch/arm/mach-omap2/Kconfig
+++ b/arch/arm/mach-omap2/Kconfig
@@ -96,6 +96,7 @@ config ARCH_OMAP2PLUS
 	select ARCH_HAS_HOLES_MEMORYMODEL
 	select ARCH_OMAP
 	select CLKSRC_MMIO
+	select GENERIC_CLOCKSOURCE_VDSO
 	select GENERIC_IRQ_CHIP
 	select GPIOLIB
 	select MACH_OMAP_GENERIC
diff --git a/arch/arm/mach-omap2/timer.c b/arch/arm/mach-omap2/timer.c
index 69c3a6d94933..bc7d177759c3 100644
--- a/arch/arm/mach-omap2/timer.c
+++ b/arch/arm/mach-omap2/timer.c
@@ -413,17 +413,14 @@ static bool use_gptimer_clksrc __initdata;
 /*
  * clocksource
  */
-static u64 clocksource_read_cycles(struct clocksource *cs)
-{
-	return (u64)__omap_dm_timer_read_counter(&clksrc,
-						     OMAP_TIMER_NONPOSTED);
-}
 
-static struct clocksource clocksource_gpt = {
-	.rating		= 300,
-	.read		= clocksource_read_cycles,
-	.mask		= CLOCKSOURCE_MASK(32),
-	.flags		= CLOCK_SOURCE_IS_CONTINUOUS,
+static struct clocksource_user_mmio clocksource_gpt = {
+	.mmio.clksrc = {
+		.rating		= 300,
+		.read		= clocksource_mmio_readl_up,
+		.mask		= CLOCKSOURCE_MASK(32),
+		.flags		= CLOCK_SOURCE_IS_CONTINUOUS,
+	},
 };
 
 static u64 notrace dmtimer_read_sched_clock(void)
@@ -505,21 +502,22 @@ static void __init omap2_gptimer_clocksource_init(int gptimer_id,
 						  const char *fck_source,
 						  const char *property)
 {
+	struct clocksource_mmio_regs mmr;
 	int res;
 
 	clksrc.id = gptimer_id;
 	clksrc.errata = omap_dm_timer_get_errata();
 
 	res = omap_dm_timer_init_one(&clksrc, fck_source, property,
-				     &clocksource_gpt.name,
+				     &clocksource_gpt.mmio.clksrc.name,
 				     OMAP_TIMER_NONPOSTED);
 
 	if (soc_is_am43xx()) {
-		clocksource_gpt.suspend = omap2_gptimer_clksrc_suspend;
-		clocksource_gpt.resume = omap2_gptimer_clksrc_resume;
+		clocksource_gpt.mmio.clksrc.suspend = omap2_gptimer_clksrc_suspend;
+		clocksource_gpt.mmio.clksrc.resume = omap2_gptimer_clksrc_resume;
 
 		clocksource_gpt_hwmod =
-			omap_hwmod_lookup(clocksource_gpt.name);
+			omap_hwmod_lookup(clocksource_gpt.mmio.clksrc.name);
 	}
 
 	BUG_ON(res);
@@ -529,12 +527,18 @@ static void __init omap2_gptimer_clocksource_init(int gptimer_id,
 				   OMAP_TIMER_NONPOSTED);
 	sched_clock_register(dmtimer_read_sched_clock, 32, clksrc.rate);
 
-	if (clocksource_register_hz(&clocksource_gpt, clksrc.rate))
+	mmr.reg_lower = clksrc.func_base + (OMAP_TIMER_COUNTER_REG & 0xff);
+	mmr.bits_lower = 32;
+	mmr.reg_upper = 0;
+	mmr.bits_upper = 0;
+	mmr.revmap = NULL;
+
+	if (clocksource_user_mmio_init(&clocksource_gpt, &mmr, clksrc.rate))
 		pr_err("Could not register clocksource %s\n",
-			clocksource_gpt.name);
+			clocksource_gpt.mmio.clksrc.name);
 	else
 		pr_info("OMAP clocksource: %s at %lu Hz\n",
-			clocksource_gpt.name, clksrc.rate);
+			clocksource_gpt.mmio.clksrc.name, clksrc.rate);
 }
 ```

{{% notice tip %}}
In some rare cases, converting the available clocksource(s) so that we
can read them directly from the vDSO might not be an option, typically
because reading them would require supervisor privileges in the CPU,
which the vDSO context excludes by definition.  For these almost desperate
situations, there is still the option for your companion core to
[intercept system calls]({{< relref "dovetail/altsched#syscall-events" >}}) to
[clock_gettime(3)](http://man7.org/linux/man-pages/man3/clock_gettime.3.html)
from the out-of-band handler, handling them directly from that spot.
This would be slower compared to a direct readout from the
vDSO, but the core would manage to get timestamps for `CLOCK_MONOTONIC`
and `CLOCK_REALTIME` clocks at least without involving the in-band stage.
EVL solves a limitation with clock sources on [legacy x86
hardware]({{< relref "core/caveat#x86-caveat" >}}) this way.
{{% /notice %}}

---

{{<lastmodified>}}
