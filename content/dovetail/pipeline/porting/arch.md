---
title: "Architecture-specific bits"
menuTitle: "Architecture bits"
date: 2018-06-27T17:07:51+02:00
weight: 20
---

## Interrupt mask virtualization

The architecture-specific code which manipulates the interrupt flag in
the CPU's state register in *arch/<your-arch>/include/asm/irqflags.h*
should be split between real and virtual interrupt control. The real
interrupt control operations are inherited from the in-band kernel
implementation. The virtual ones should be built upon services
provided by the [interrupt pipeline core]({{%relref
"dovetail/pipeline/porting/irqflow.md" %}}).

+ firstly, the original *arch\_local_*\* helpers should be renamed as
*native_*\* helpers, affecting the hardware interrupt state in the
CPU. This naming convention is imposed on the architecture code by the
generic helpers in _include/asm-generic/irq\_pipeline.h_.

> Example: introducing the *native* interrupt state accessors for the
  ARM architecture

```
--- a/arch/arm/include/asm/irqflags.h
+++ b/arch/arm/include/asm/irqflags.h
 #if __LINUX_ARM_ARCH__ >= 6
 
 #define arch_local_irq_save arch_local_irq_save
-static inline unsigned long arch_local_irq_save(void)
+static inline unsigned long native_irq_save(void)
 {
 	unsigned long flags;
 
 	asm volatile(
-		"	mrs	%0, " IRQMASK_REG_NAME_R "	@ arch_local_irq_save\n"
+		"	mrs	%0, " IRQMASK_REG_NAME_R "	@ native_irq_save\n"
 		"	cpsid	i"
 		: "=r" (flags) : : "memory", "cc");
 	return flags;
 }
 
 #define arch_local_irq_enable arch_local_irq_enable
-static inline void arch_local_irq_enable(void)
+static inline void native_irq_enable(void)
 {
 	asm volatile(
-		"	cpsie i			@ arch_local_irq_enable"
+		"	cpsie i			@ native_irq_enable"
 		:
 		:
 		: "memory", "cc");
 }
 
 #define arch_local_irq_disable arch_local_irq_disable
-static inline void arch_local_irq_disable(void)
+static inline void native_irq_disable(void)
 {
 	asm volatile(
-		"	cpsid i			@ arch_local_irq_disable"
+		"	cpsid i			@ native_irq_disable"
 		:
 		:
 		: "memory", "cc");
@@ -69,12 +76,12 @@ static inline void arch_local_irq_disable(void)
  * Save the current interrupt enable state & disable IRQs
  */
 #define arch_local_irq_save arch_local_irq_save
-static inline unsigned long arch_local_irq_save(void)
+static inline unsigned long native_irq_save(void)
 {
 	unsigned long flags, temp;
 
 	asm volatile(
-		"	mrs	%0, cpsr	@ arch_local_irq_save\n"
+		"	mrs	%0, cpsr	@ native_irq_save\n"
 		"	orr	%1, %0, #128\n"
 		"	msr	cpsr_c, %1"
 		: "=r" (flags), "=r" (temp)
@@ -87,11 +94,11 @@ static inline unsigned long arch_local_irq_save(void)
  * Enable IRQs
  */
 #define arch_local_irq_enable arch_local_irq_enable
-static inline void arch_local_irq_enable(void)
+static inline void native_irq_enable(void)
 {
 	unsigned long temp;
 	asm volatile(
-		"	mrs	%0, cpsr	@ arch_local_irq_enable\n"
+		"	mrs	%0, cpsr	@ native_irq_enable\n"
 		"	bic	%0, %0, #128\n"
 		"	msr	cpsr_c, %0"
 		: "=r" (temp)
@@ -103,11 +110,11 @@ static inline void arch_local_irq_enable(void)
  * Disable IRQs
  */
 #define arch_local_irq_disable arch_local_irq_disable
-static inline void arch_local_irq_disable(void)
+static inline void native_irq_disable(void)
 {
 	unsigned long temp;
 	asm volatile(
-		"	mrs	%0, cpsr	@ arch_local_irq_disable\n"
+		"	mrs	%0, cpsr	@ native_irq_disable\n"
 		"	orr	%0, %0, #128\n"
 		"	msr	cpsr_c, %0"
 		: "=r" (temp)
@@ -153,11 +160,11 @@ static inline void arch_local_irq_disable(void)
  * Save the current interrupt enable state.
  */
 #define arch_local_save_flags arch_local_save_flags
-static inline unsigned long arch_local_save_flags(void)
+static inline unsigned long native_save_flags(void)
 {
 	unsigned long flags;
 	asm volatile(
-		"	mrs	%0, " IRQMASK_REG_NAME_R "	@ local_save_flags"
+		"	mrs	%0, " IRQMASK_REG_NAME_R "	@ native_save_flags"
 		: "=r" (flags) : : "memory", "cc");
 	return flags;
 }
@@ -166,21 +173,28 @@ static inline unsigned long arch_local_save_flags(void)
  * restore saved IRQ & FIQ state
  */
 #define arch_local_irq_restore arch_local_irq_restore
-static inline void arch_local_irq_restore(unsigned long flags)
+static inline void native_irq_restore(unsigned long flags)
 {
 	asm volatile(
-		"	msr	" IRQMASK_REG_NAME_W ", %0	@ local_irq_restore"
+		"	msr	" IRQMASK_REG_NAME_W ", %0	@ native_irq_restore"
 		:
 		: "r" (flags)
 		: "memory", "cc");
 }
 
 #define arch_irqs_disabled_flags arch_irqs_disabled_flags
-static inline int arch_irqs_disabled_flags(unsigned long flags)
+static inline int native_irqs_disabled_flags(unsigned long flags)
 {
 	return flags & IRQMASK_I_BIT;
 }
 
+static inline bool native_irqs_disabled(void)
+{
+	unsigned long flags = native_save_flags();
+	return native_irqs_disabled_flags(flags);
+}
+
+#include <asm/irq_pipeline.h>
 #include <asm-generic/irqflags.h>
 
 #endif /* ifdef __KERNEL__ */
```

+ finally, a new set of *arch\_local\_*\* helpers should be provided,
  affecting the virtual interrupt disable flag implemented by the
  pipeline core for controlling the in-band stage protection against
  interrupts. It is good practice to implement this set in a separate
  file available for inclusion from *\<asm/irq_pipeline.h>*.

> Example: providing the virtual interrupt state accessors for the ARM
  architecture
```
--- /dev/null
+++ b/arch/arm/include/asm/irq_pipeline.h
@@ -0,0 +1,138 @@
+#ifndef _ASM_ARM_IRQ_PIPELINE_H
+#define _ASM_ARM_IRQ_PIPELINE_H
+
+#include <asm-generic/irq_pipeline.h>
+
+#ifdef CONFIG_IRQ_PIPELINE
+
+static inline notrace unsigned long arch_local_irq_save(void)
+{
+	int stalled = inband_irq_save();
+	barrier();
+	return arch_irqs_virtual_to_native_flags(stalled);
+}
+
+static inline notrace void arch_local_irq_enable(void)
+{
+	barrier();
+	inband_irq_enable();
+}
+
+static inline notrace void arch_local_irq_disable(void)
+{
+	inband_irq_disable();
+	barrier();
+}
+
+static inline notrace unsigned long arch_local_save_flags(void)
+{
+	int stalled = inband_irqs_disabled();
+	barrier();
+	return arch_irqs_virtual_to_native_flags(stalled);
+}
+
+static inline int arch_irqs_disabled_flags(unsigned long flags)
+{
+	return native_irqs_disabled_flags(flags);
+}
+
+static inline notrace void arch_local_irq_restore(unsigned long flags)
+{
+	if (!arch_irqs_disabled_flags(flags))
+		__inband_irq_enable();
+	barrier();
+}
+
+#else /* !CONFIG_IRQ_PIPELINE */
+
+static inline unsigned long arch_local_irq_save(void)
+{
+	return native_irq_save();
+}
+
+static inline void arch_local_irq_enable(void)
+{
+	native_irq_enable();
+}
+
+static inline void arch_local_irq_disable(void)
+{
+	native_irq_disable();
+}
+
+static inline unsigned long arch_local_save_flags(void)
+{
+	return native_save_flags();
+}
+
+static inline void arch_local_irq_restore(unsigned long flags)
+{
+	native_irq_restore(flags);
+}
+
+static inline int arch_irqs_disabled_flags(unsigned long flags)
+{
+	return native_irqs_disabled_flags(flags);
+}
+
+#endif /* !CONFIG_IRQ_PIPELINE */
+
+#endif /* _ASM_ARM_IRQ_PIPELINE_H */
```

{{% notice note %}}
This new file should include *\<asm-generic/irq_pipeline.h>* early to
get access to the pipeline declarations it needs. This inclusion
should be unconditional, even if the kernel is built with
CONFIG_IRQ_PIPELINE disabled.
{{% /notice %}}

### Providing support for merged interrupt states

The generic interrupt pipeline implementation requires the arch-level
support code to provide for a pair of helpers aimed at translating the
[virtual interrupt disable flag]({{%relref
"dovetail/pipeline/optimistic.md#virtual-i-flag" %}}) to the interrupt bit in
the CPU's status register (e.g. PSR_I_BIT for ARM) and
conversely. These helpers are used to create combined state words
merging the virtual and real interrupt states.

+ `arch_irqs_virtual_to_native_flags(int stalled)` must return a long
  word remapping the boolean value of @stalled to the CPU's interrupt
  bit position in the status register. All other bits must be
  cleared.

  - On ARM, this can be expressed as `(stalled ? PSR_I_BIT : 0)`.
  - on x86, that would rather be `(stalled ? 0 : X86_EFLAGS_IF)`.

+ `arch_irqs_native_to_virtual_flags(unsigned long flags)` must return
  a long word remapping the CPU's interrupt bit in @flags to an
  arbitrary bit position, choosen not to conflict with the former.  In
  other words, the CPU's interrupt state bit received in @flags should
  be shifted to a free position picked arbitrarily in the return
  value. All other bits must be cleared.

  - On ARM, using bit position 31 to reflect the virtual state, this
    is expressed as `(hard_irqs_disabled_flags(flags) ? (1 << 31) : 0)`.

  - On any other architecture, the implementation would be similar,
    using whatever bit position is available which would not conflict
    with the CPU's interrupt bit position.

```
--- a/arch/arm/include/asm/irqflags.h
+++ b/arch/arm/include/asm/irqflags.h
/*
 * CPU interrupt mask handling.
 */
#ifdef CONFIG_CPU_V7M
 #define IRQMASK_REG_NAME_R "primask"
 #define IRQMASK_REG_NAME_W "primask"
 #define IRQMASK_I_BIT	1
+#define IRQMASK_I_POS	0
 #else
 #define IRQMASK_REG_NAME_R "cpsr"
 #define IRQMASK_REG_NAME_W "cpsr_c"
 #define IRQMASK_I_BIT	PSR_I_BIT
+#define IRQMASK_I_POS	7
 #endif
+#define IRQMASK_i_POS	31
```
{{% notice tip %}}
`IRQMASK_i_POS` (note the minus 'i') is the free bit position in the
combo word where the ARM port stores the original CPU's interrupt
state in the combo word. This position can't conflict with `IRQMASK_I_POS`,
which is an alias to `PSR_I_BIT` (bit position 0 or 7).
{{% /notice %}}
```
--- /dev/null
+++ b/arch/arm/include/asm/irq_pipeline.h
+
+static inline notrace
+unsigned long arch_irqs_virtual_to_native_flags(int stalled)
+{
+	return (!!stalled) << IRQMASK_I_POS;
+}
+static inline notrace
+unsigned long arch_irqs_native_to_virtual_flags(unsigned long flags)
+{
+	return (!!hard_irqs_disabled_flags(flags)) << IRQMASK_i_POS;
+}
```

{{% notice info %}}
Once all of these changes are in, the generic helpers from
<linux/irqflags.h> such as `local_irq_disable()` and
`local_irq_enable()` actually refer to the **virtual** protection
scheme when interrupts are pipelined, which eventually allows to
implement [interrupt deferral]({{%relref
"dovetail/pipeline/optimistic.md" %}}) for the protected in-band code running
over the in-band stage.
{{% /notice %}}
    
## Adapting the assembly code to IRQ pipelining {#arch-irq-handling}

### Interrupt entry {#arch-irq-entry}

As generic IRQ handling [is a requirement]({{%relref
"dovetail/pipeline/porting/prerequisites.md" %}}) for supporting
Dovetail, the low-level interrupt handler living in the assembly
portion of the architecture code can still deliver all interrupt
events to the original C handler provided by the _irqchip_
driver. That handler should in turn invoke:

- `handle_domain_irq()` for parent device IRQs

- `generic_handle_irq()` for cascaded device IRQs (decoded from the
  parent handler)

For those routines, the initial task of inserting an interrupt at the
head of the pipeline is directly handled from the _genirq_ layer they
belong to. This means that there is usually not much to do other than
making a quick check in the implementation of the parent IRQ handler
in the relevant *irqchip* driver, applying the [rules of
thumb]({{%relref "dovetail/rulesofthumb.md" %}}) carefully.

{{% notice tip %}}
On some ARM platform equipped with a fairly common GIC controller,
that would mean inspecting the function `gic_handle_irq()` for
instance.
{{% /notice %}}

- the arch-specific `handle_IPI()` or equivalent for special
  inter-processor interrupts

IPIs must be dealt with by [specific changes]({{%relref
"dovetail/pipeline/porting/arch.md#dealing-with-ipis" %}}) introduced
by the port we will cover later.

### Interrupt exit {#arch-irq-exit}

When interrupt pipelining is disabled, the kernel normally runs an
epilogue after each interrupt or exception event was handled. If the
event happened while the CPU was running some kernel code, the
epilogue would check for a potential rescheduling opportunity in case
`CONFIG_PREEMPT` is enabled. If a user-space task was preempted by the
event, additional conditions would be checked for such as a signal
pending delivery for that task.

Because interrupts are only virtually masked for the in-band code when
pipelining is enabled, IRQs can still be taken by the CPU and passed
on to the low-level assembly handlers, so that they can enter the
interrupt pipeline.

> Running the regular epilogue afer an IRQ is valid only if the kernel
was actually accepting interrupts when the event happened (i.e. the
virtual interrupt disable flag was clear), and running in-band code.

In all other cases, except for the interrupt pipeline core, the rest
of the kernel does _not_ expect those IRQs to ever happen in the first
place. Therefore, running the epilogue in such circumstances would be
at odds with the kernel's logic.  In addition, low-level handlers must
have been made aware that they might receive an event under such
conditions.

For instance, the original ARM code for handling an IRQ which has
preempted a kernel context would look like this:

```
__irq_svc:
	svc_entry
	irq_handler

#ifdef CONFIG_PREEMPT
	ldr	r8, [tsk, #TI_PREEMPT]		@ get preempt count
	ldr	r0, [tsk, #TI_FLAGS]		@ get flags
	teq	r8, #0				@ if preempt count != 0
	movne	r0, #0				@ force flags to 0
	tst	r0, #_TIF_NEED_RESCHED
	blne	svc_preempt
#endif
```

In order to properly handle interrupts in a pipelined delivery model,
we have to detect whether the in-band kernel was ready to receive such
event, acting upon it accordingly. To this end, the ARM port passes
the event to a trampoline routine instead
(`handle_arch_irq_pipelined()`), expecting on return a decision
whether or not the epilogue code should run next. In the illustration
below, this decision is returned as a boolean status to the caller,
non-zero meaning that we may run the epilogue, zero otherwise.

```
--- a/arch/arm/kernel/entry-armv.S
+++ b/arch/arm/kernel/entry-armv.S
 	.macro	irq_handler
 #ifdef CONFIG_MULTI_IRQ_HANDLER
-	ldr	r1, =handle_arch_irq
 	mov	r0, sp
 	badr	lr, 9997f
+#ifdef CONFIG_IRQ_PIPELINE
+	ldr	r1, =handle_arch_irq_pipelined
+	mov	pc, r1
+#else	
+	ldr	r1, =handle_arch_irq
 	ldr	pc, [r1]
-#else
+#endif
+#elif CONFIG_IRQ_PIPELINE
+#error "Legacy IRQ handling not pipelined"
+#else	
 	arch_irq_handler_default
 #endif
 9997:
 	.endm
```

The trampoline routine added to the original code, first delivers the
interrupt to the machine-defined handler, then tells the caller
whether the regular epilogue may run for such event.

```
--- a/arch/arm/kernel/irq.c
+++ b/arch/arm/kernel/irq.c
@@ -112,6 +112,15 @@ void __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
 }
 #endif
 
+#ifdef CONFIG_IRQ_PIPELINE
+asmlinkage int __exception_irq_entry
+handle_arch_irq_pipelined(struct pt_regs *regs)
+{
+	handle_arch_irq(regs);
+	return running_inband() && !irqs_disabled();
+}
+#endif
+
```

Eventually, the low-level assembly handler receiving the interrupt
event is adapted, in order to carry out the earlier decision by
`handle_arch_irq_pipelined()`, skipping the epilogue code if required
to.


```
--- a/arch/arm/kernel/entry-armv.S
+++ b/arch/arm/kernel/entry-armv.S
 __irq_svc:
 	svc_entry
 	irq_handler
+#ifdef CONFIG_IRQ_PIPELINE
+	tst	r0, r0
+	beq	1f
+#endif
 
 #ifdef CONFIG_PREEMPT
 	ldr	r8, [tsk, #TI_PREEMPT]		@ get preempt count
 	blne	svc_preempt
 #endif
 
+1:
 	svc_exit r5, irq = 1			@ return from exception
```

{{% notice warning %}}
Taking the fast exit path when applicable is critical to the stability of the
target system to prevent [invalid re-entry]({{%relref
"dovetail/pipeline/optimistic.md#no-inband-reentry" %}}) of the in-band kernel
code.
{{% /notice %}}

### Fault exit

Similarly to the interrupt exit case, the low-level fault handling
code must skip the epilogue code when the fault was taken over an
out-of-band context. Upon fault, the current interrupt state is not
considered for determining whether we should run the epilogue, since
a fault may occur independently of such state.

> Running the regular epilogue after a fault is valid only if that
fault was triggered by some in-band code, excluding any fault raised
by out-of-band code.

For instance, the original ARM code for returning from an exception
event would be modified as follows:

```
--- a/arch/arm/kernel/entry-armv.S
+++ b/arch/arm/kernel/entry-armv.S
@@ -754,7 +772,7 @@ ENTRY(ret_from_exception)
  UNWIND(.cantunwind	)
 	get_thread_info tsk
 	mov	why, #0
-	b	ret_to_user
+	ret_to_user_pipelined r1
  UNWIND(.fnend		)
```

With the implementation of `ret_to_user_pipelined` checking for the
current stage, skipping the epilogue if the faulting code was running
over an out-of-band context:

```
--- a/arch/arm/kernel/entry-header.S
+++ b/arch/arm/kernel/entry-header.S
+/*
+ * Branch to the exception epilogue, skipping the in-band work
+ * if running over the oob interrupt stage.
+ */
+	.macro ret_to_user_pipelined, tmp
+#ifdef CONFIG_IRQ_PIPELINE
+	ldr	\tmp, [tsk, #TI_LOCAL_FLAGS]
+	tst	\tmp, #_TLF_OOB
+	bne	fast_ret_to_user
+#endif
+	b	ret_to_user
+	.endm
+
```

`_TLF_OOB` is a local `thread_info` flag denoting a current task
running out-of-band code over the oob stage. If set, the epilogue must
be skipped.

## Reconciling the virtual interrupt state to the epilogue logic

A tricky issue to address when pipelining interrupts is about making
sure that the logic from the epilogue routine
(e.g. _do\_work\_pending()_, _do\_notify\_resume()_) actually runs in
the expected [(virtual) interrupt state]({{%relref
"dovetail/pipeline/optimistic.md#virtual-i-flag" %}}) for the in-band stage.

Reconciling the virtual interrupt state to the in-band logic dealing
with interrupts is required because in a pipelined interrupt model,
the virtual interrupt state of the in-band stage does not necessarily
reflect the CPU's interrupt state on entry to the early assembly code
handling the IRQ events. Typically, a CPU would always automatically
disable interrupts hardware-wise when taking an IRQ, which may
contradict the software-managed virtual state until both are
eventually reconciled.

Those rules of thumb should be kept in mind when adapting the epilogue
routine to interrupt pipelining:

- most often, such routine is supposed to be entered with (hard)
  interrupts off when called from the assembly code which handles
  kernel entry/exit transitions (e.g. arch/arm/kernel/entry-common.S).
  Therefore, this routine may have to reconcile the virtual interrupt
  state with such expectation, since according to the [interrupt exit
  rules]({{%relref "dovetail/pipeline/porting/arch.md#arch-irq-exit" %}}) we
  discussed earlier, such state has to be originally enabled (i.e. the
  in-band stall bit is clear) for the epilogue code to run in the
  first place.

- conversely, we must keep the hard interrupt state consistent upon
  return from the epilogue code with the one received on
  entry. Typically, hard interrupts must be disabled before leaving
  this code if we entered it that way.

- likewise, we must also keep the virtual interrupt state consistent
  upon return of the epilogue code with the one received on entry. In
  other words, the stall bit of the in-band stage must be restored to its
  original state on entry before leaving this code.

- _schedule()_ must be called with interrupts virtually disabled for
  the in-band stage, but the CPU's interrupt state should allow for
  IRQs to be taken in order to minimize latency for the oob stage.

- generally speaking, while we may need the in-band stage to be
  stalled when the in-band kernel code expects this, we still want
  most of the epilogue code to run with [hard interrupts
  enabled]({{%relref
  "dovetail/pipeline/usage/interrupt_protection.md#hard-irq-protection"%}})
  to shorten the interrupt latency for the oob stage, where autonomous
  cores live.

> Reconciling the interrupt state on ARM64 epilogue
```
 #include <linux/errno.h>
 #include <linux/kernel.h>
 #include <linux/signal.h>
+#include <linux/irq_pipeline.h>
 #include <linux/personality.h>
 #include <linux/freezer.h>
 #include <linux/stddef.h>
@@ -915,24 +916,34 @@ static void do_signal(struct pt_regs *regs)
 asmlinkage void do_notify_resume(struct pt_regs *regs,
 				 unsigned long thread_flags)
 {
+	WARN_ON_ONCE(irq_pipeline_debug() &&
+		(irqs_disabled() || running_oob()));
+
 	/*
 	 * The assembly code enters us with IRQs off, but it hasn't
 	 * informed the tracing code of that for efficiency reasons.
 	 * Update the trace code with the current status.
 	 */
-	trace_hardirqs_off();
+	if (!irqs_pipelined())
+		trace_hardirqs_off();
 
 	do {
+		if (irqs_pipelined())
+			local_irq_disable();
+
 		/* Check valid user FS if needed */
 		addr_limit_user_check();
 
 		if (thread_flags & _TIF_NEED_RESCHED) {
 			/* Unmask Debug and SError for the next task */
-			local_daif_restore(DAIF_PROCCTX_NOIRQ);
+			local_daif_restore(irqs_pipelined() ?
+					DAIF_PROCCTX : DAIF_PROCCTX_NOIRQ);
 
 			schedule();
 		} else {
 			local_daif_restore(DAIF_PROCCTX);
+			if (irqs_pipelined())
+				local_irq_enable();
 
 			if (thread_flags & _TIF_UPROBE)
 				uprobe_notify_resume(regs);
@@ -950,9 +961,17 @@ asmlinkage void do_notify_resume(struct pt_regs *regs,
 				fpsimd_restore_current_state();
 		}
 
+		/*
+		 * CAUTION: we may have restored the fpsimd state for
+		 * current with no other opportunity to check for
+		 * _TIF_FOREIGN_FPSTATE until we are back running on
+		 * el0, so we must not take any interrupt until then,
+		 * otherwise we may end up resuming with some OOB
+		 * thread's fpsimd state.
+		 */
 		local_daif_mask();
 		thread_flags = READ_ONCE(current_thread_info()->flags);
-	} while (thread_flags & _TIF_WORK_MASK);
+	} while (inband_irq_pending() || (thread_flags & _TIF_WORK_MASK));
 }
```

## Dealing with IPIs {#dealing-with-ipis}

Although the pipeline does not directly use inter-processor interrupts
internally, it provides a simple API to autonomous cores for
implementing [IPI-based messaging]({{< relref
"dovetail/pipeline/usage/pipeline_inject.md#oob-ipi" >}}) between
CPUs.  This feature requires the Dovetail port to implement a few bits
of architecture-specific code. The arch-specific Dovetail
implementation must provide support for two operations:

- sending the additional IPI signals defined by the [Dovetail API]({{<
  relref "dovetail/pipeline/usage/pipeline_inject.md#oob-ipi" >}})
  using the mechanism available from your hardware for inter-processor
  messaging, upon request from `irq_pipeline_send_remote()`.

- [dispatching deferred IPIs]({{< relref
  "dovetail/pipeline/porting/irqflow.md#arch-do-irq" >}}) to their
  respective in-band handler upon request from the Dovetail core.

### Mapping IPI numbers to pipelined IRQ numbers

Without Dovetail, Linux architecture ports do not assign interrupt
descriptors (i.e. `struct irq_desc`) to IPIs. Instead, IPI receipt is
handled separately from device IRQs, where those events are not
channeled through the common generic IRQ layer but rather immediately
handled by arch-specific code. With Dovetail, we do generally need
IPIs to have a valid interrupt descriptor, so that we can request the
out-of-band IPIs using the [generic IRQ API]({{< relref
"dovetail/pipeline/usage/irq_handling.md" >}}), exactly as we can
request device interrupts.

The easiest way to achieve this mapping is to create a new interrupt
domain for IPIs, which are in essence per-CPU events. Interrupts from
this domain need no [generic flow handling code]({{< relref
"dovetail/pipeline/porting/irqflow.md#irqchip-fixup" >}}) since this
part is directly handled from the arch-specific code, therefore a
dummy interrupt chip controller can be associated to this domain. For
instance, the ARM implementation installs all IPIs in such a domain,
from IRQ2048 (`OOB_IPI_BASE`) and on. Since the arch-specific
implementation of `handle_IPI()` in a Dovetail-enabled kernel should
generate a interrupt from the IPI domain by software each time we
receive an IPI event from the hardware, `handle_synthetic_irq()` is
the proper flow handler for IPI interrupts.

> IPI domain for ARM
```
static struct irq_domain *sipic_domain;

static void sipic_irq_noop(struct irq_data *data) { }

static unsigned int sipic_irq_noop_ret(struct irq_data *data)
{
	return 0;
}

static struct irq_chip sipic_chip = {
	.name		= "SIPIC",
	.irq_startup	= sipic_irq_noop_ret,
	.irq_shutdown	= sipic_irq_noop,
	.irq_enable	= sipic_irq_noop,
	.irq_disable	= sipic_irq_noop,
	.irq_ack	= sipic_irq_noop,
	.irq_mask	= sipic_irq_noop,
	.irq_unmask	= sipic_irq_noop,
	.flags		= IRQCHIP_PIPELINE_SAFE | IRQCHIP_SKIP_SET_WAKE,
};

static int sipic_irq_map(struct irq_domain *d, unsigned int irq,
			irq_hw_number_t hwirq)
{
	irq_set_percpu_devid(irq);
	irq_set_chip_and_handler(irq, &sipic_chip, handle_synthetic_irq);

	return 0;
}

static struct irq_domain_ops sipic_domain_ops = {
	.map	= sipic_irq_map,
};

static void create_ipi_domain(void)
{
	/*
	 * Create an IRQ domain for mapping all IPIs (in-band and
	 * out-of-band), with fixed sirq numbers starting from
	 * OOB_IPI_BASE. The sirqs obtained can be injected into the
	 * pipeline upon IPI receipt like other interrupts.
	 */
	sipic_domain = irq_domain_add_simple(NULL, NR_IPI + OOB_NR_IPI,
					     OOB_IPI_BASE,
					     &sipic_domain_ops, NULL);
}

void __init arch_irq_pipeline_init(void)
{
#ifdef CONFIG_SMP
	create_ipi_domain();
#endif
}
```

{{% notice note %}}
Initializing the IPI domain should be done from
the `arch_irq_pipeline_init()` handler, which Dovetail calls while
setting up the interrupt pipelining machinery early at kernel boot.
{{% /notice %}}

### Short of IPI vectors? Multiplex!

The main issue you may face with these requirements stands in the
limitation your hardware may have with respect to the number of
distinct IPI signals available from the interrupt
controller. Typically, the ARM generic interrupt controller (aka
_GIC_) available with the Cortex CPU series provides 16 distinct IPIs,
half of which should be reserved to the firmware, which leaves only 8
IPIs available to the kernel. Since all of them are already in use for
in-band work in the mainline implementation, we are short of available
IPI vectors for adding the two additional signals we need.  For this
reason, the Dovetail ports for ARM and ARM64 have reshuffled the way
IPI signaling is implemented.

Before this rework, a 1:1 mapping exists between the logical IPI
numbers used by the kernel to refer to inter-processor messages, and
the physical, so-called _SGI_ numbers which stands for [Software
Generated
Interrupts](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/CEGIDCIC.html):

|      IPI Message     |        Logical IPI number            |  Physical SGI number |
| :------------------: | :----------------------------------: | :------------------: |
|   IPI_WAKEUP          |   0  | 0  |
|   IPI_TIMER           |   1  | 1  |
|   IPI_RESCHEDULE      |   2  | 2  |
|   IPI_CALL_FUNC       |   3  | 3  |
|   IPI_CPU_STOP        |   4  | 4  |
|   IPI_IRQ_WORK        |   5  | 5  |
|   IPI_COMPLETION      |   6  | 6  |
|   IPI_CPU_BACKTRACE   |   7  | 7  |

After the Dovetail rework, all pre-existing in-band IPIs are now
multiplexed over SGI0, which leaves seven SGIs available for adding
the out-of-band IPI messages we need, from which we only need two. The
resulting mapping is as follows:

|      IPI Message     |        Logical IPI number            |  Physical SGI number | Pipelined IRQ number |
| :------------------: | :----------------------------------: | :------------------: |:------------------: |
|   IPI_WAKEUP          |   0  | 0  | 2048  |
|   IPI_TIMER           |   1  | 0  | 2049  |
|   IPI_RESCHEDULE      |   2  | 0  | 2050  |
|   IPI_CALL_FUNC       |   3  | 0  | 2051  |
|   IPI_CPU_STOP        |   4  | 0  | 2052  |
|   IPI_IRQ_WORK        |   5  | 0  | 2053  |
|   IPI_COMPLETION      |   6  | 0  | 2054  |
|   IPI_CPU_BACKTRACE   |   7  | 0  | 2055  |
|   TIMER_OOB_IPI       |   8  | 1  | 2056  |
|   RESCHEDULE_OOB_IPI  |   9  | 2  | 2057  |

The implementation of the IPI multiplexing for ARM takes place in
_arch/arm/kernel/smp.c_. The logic - as illustrated below - is fairly
straightforward:

- on the issuer side, if the IPI we need to send belongs to the
  in-band set (`ipinr < NR_IPI`), log the pending signal into a
  per-CPU global bitmask (`ipi_messages`) then issue SGI0. Otherwise,
  issue either SGI1 or SGI2 for signaling the corresponding
  out-of-band IPIs directly.

- upon receipt, if we received SGI0, iterate over the pending in-band
  IPIs by reading the per-CPU bitmask (`ipi_messages`) demultiplexing
  the logical IPI numbers as we go before pushing the corresponding
  IRQ event to the pipeline entry (`generic_pipeline_irq()`). If SGI1
  or SGI2 were received instead, remap the incoming signal to either
  TIMER_OOB_IPI or RESCHEDULE_OOB_IPI before feeding the pipeline with
  the corresponding IRQ event, which is either 2056 or 2057.

```
static DEFINE_PER_CPU(unsigned long, ipi_messages);

static inline
void send_IPI_message(const struct cpumask *target, unsigned int ipinr)
{
	unsigned int cpu, sgi;

	if (ipinr < NR_IPI) {
		/* regular in-band IPI (multiplexed over SGI0). */
		trace_ipi_raise_rcuidle(target, ipi_types[ipinr]);
		for_each_cpu(cpu, target)
			set_bit(ipinr, &per_cpu(ipi_messages, cpu));
		smp_mb();
		sgi = 0;
	} else	/* out-of-band IPI (SGI1-3). */
		sgi = ipinr - NR_IPI + 1;

	__smp_cross_call(target, sgi);
}

static inline
void handle_IPI_pipelined(int sgi, struct pt_regs *regs)
{
	unsigned int ipinr, irq;
	unsigned long *pmsg;

	if (sgi) {		/* SGI1-3 */
		irq = sgi + NR_IPI - 1 + OOB_IPI_BASE;
		generic_pipeline_irq(irq, regs);
		return;
	}

	/* In-band IPI (0..NR_IPI - 1) multiplexed over SGI0. */
	pmsg = raw_cpu_ptr(&ipi_messages);
	while (*pmsg) {
		ipinr = ffs(*pmsg) - 1;
		clear_bit(ipinr, pmsg);
		irq = OOB_IPI_BASE + ipinr;
		generic_pipeline_irq(irq, regs);
	}
}
```

As illustrated in the example of [in-band delivery glue code]({{<
relref "dovetail/pipeline/porting/irqflow.md#arch-do-irq" >}}), the
ARM ports distinguishes between device IRQs and IPIs based on the
pipelined IRQ number, with anything in the range
[OOB_IPI_BASE..OOB_IPI_BASE + 10] being dispatched as an IPI to the
`__handle_IPI()` routine.

Since out-of-band IPI messages are supposed to be exclusively handled
by [out-of-band handlers]({{< relref
"dovetail/pipeline/usage/irq_handling.md" >}}), `__handle_IPI()` is
not required to handle them specifically. However, this is good
practice to detect them, at least for debugging purpose during the
early stage of the Dovetail port.
