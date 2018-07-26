---
title: "Architecture-specific bits"
menuTitle: "Architecture bits"
date: 2018-06-27T17:07:51+02:00
weight: 3
draft: false
---

## Interrupt mask virtualization

The architecture-specific code which manipulates the interrupt flag in
the CPU's state register in *arch/<your-arch>/include/asm/irqflags.h*
should be split between real and virtual interrupt control. The real
interrupt control operations are inherited from the in-band kernel
implementation. The virtual ones should be built upon services
provided by the [interrupt pipeline core]({{%relref
"pipeline/porting/irqflow.md" %}}).

+ firstly, the original *arch\_local_*\* helpers should be renamed as
*native_*\* helpers, affecting the hardware interrupt state in the
CPU.

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
  pipeline core for controlling the root stage protection against
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
+	int stalled = root_irq_save();
+	barrier();
+	return arch_irqs_virtual_to_native_flags(stalled);
+}
+
+static inline notrace void arch_local_irq_enable(void)
+{
+	barrier();
+	root_irq_enable();
+}
+
+static inline notrace void arch_local_irq_disable(void)
+{
+	root_irq_disable();
+	barrier();
+}
+
+static inline notrace unsigned long arch_local_save_flags(void)
+{
+	int stalled = root_irqs_disabled();
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
+		__root_irq_enable();
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
"pipeline/optimistic.md#virtual-i-flag" %}}) to the interrupt bit in
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
"pipeline/optimistic.md" %}}) for the protected in-band code running
over the root stage.
{{% /notice %}}
    
## Adapting the assembly code to IRQ pipelining {#arch-irq-handling}

### Interrupt entry

As generic IRQ handling [is a requirement]({{%relref
"pipeline/porting/prerequisites.md" %}}) for supporting Dovetail, the
low-level interrupt handler living in the assembly portion of the
architecture code can still deliver all interrupt events to the
original C handler provided by the _irqchip_ driver. That handler
should in turn invoke:

- `handle_domain_irq()` for parent device IRQs

- `generic_handle_irq()` for cascaded device IRQs (decoded from the
  parent handler)

For those routines, the initial task of inserting an interrupt at the
head of the pipeline is directly handled from the _genirq_ layer they
belong to. This means that there is usually not much to do other than
making a quick check in the implementation of the parent IRQ handler,
in the relevant *irqchip* driver, applying the [rules of
thumb]({{%relref "pipeline/rulesofthumb.md" %}}) carefully.

{{% notice tip %}}
On some ARM platform equipped with a fairly common GIC controller,
that would mean inspecting the function `gic_handle_irq()` for
instance.
{{% /notice %}}

- the arch-specific `handle_IPI()` or equivalent for special
  inter-processor interrupts

IPIs must be dealt with by [specific changes]({{%relref
"pipeline/porting/arch.md#dealing-with-ipis" %}}) introduced by the
port we will cover later.

### Interrupt exit

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
+	return on_root_stage() && !irqs_disabled();
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
"pipeline/optimistic.md#no-inband-reentry" %}}) of the in-band kernel
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
+ * if running over the head interrupt stage.
+ */
+	.macro ret_to_user_pipelined, tmp
+#ifdef CONFIG_IRQ_PIPELINE
+	ldr	\tmp, [tsk, #TI_LOCAL_FLAGS]
+	tst	\tmp, #_TLF_HEAD
+	bne	fast_ret_to_user
+#endif
+	b	ret_to_user
+	.endm
+
```

`_TLF_HEAD` is a local `thread_info` flag denoting a current task
running out-of-band code over the head stage. If set, the epilogue
must be skipped.

## Dealing with IPIs {#dealing-with-ipis}

Support for inter-processor interrupts is architecture-specific code.

TBC.
