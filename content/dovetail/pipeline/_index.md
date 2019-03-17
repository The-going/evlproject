---
title: "Interrupt pipeline"
date: 2018-06-27T08:32:51+02:00
weight: 10
pre: "&rsaquo; "
---

The autonomous core has to act upon device interrupts with no delay,
regardless of the other kernel operations which may be ongoing when
the interrupt is received by the CPU. Therefore, there is a basic
requirement for prioritizing interrupt masking and delivery between
the autonomous core and GPOS operations, while maintaining consistent
internal serialization for the kernel.

However, to protect from deadlocks and maintain data integrity, Linux
hard disables interrupts around any critical section of code which
must not be preempted by interrupt handlers on the same CPU, enforcing
a strictly serialized execution among those contexts. The
unpredictable delay this may cause before external events can be
handled is a major roadblock for kernel components requiring
predictable and very short response times to external events, in the
range of a few microseconds.

To address this issue, Dovetail introduces a mechanism called
*interrupt pipelining* which turns all device IRQs into pseudo-NMIs,
only to run NMI-safe interrupt handlers from the perspective of the
main kernel activities. This is achieved by substituting real
interrupt masking in a CPU by a software-based, virtual interrupt
masking when the in-band stage is active on such CPU. This way, the
autonomous core can receive IRQs as long as it did not mask interrupts
in the CPU, regardless of the virtual interrupt state maintained by
the in-band side. Dovetail monitors the virtual state to decide when
IRQ events should be allowed to flow down to the in-band stage where
the main kernel executes. This way, the assumptions the in-band code
makes about running interrupt-free or not are still valid.

## Two-stage IRQ pipeline {#two-stage-pipeline}

Interrupt pipelining is a lightweight approach based on the
introduction of a separate, high-priority execution stage for running
out-of-band interrupt handlers immediately upon IRQ receipt, which
cannot be delayed by the in-band, main kernel work. By immediately,
we mean unconditionally, regardless of whether the in-band kernel code
had disabled interrupts when the event arrived, using the common
`local_irq_save()`, `local_irq_disable()` helpers or any of their
derivatives.  IRQs which have no handlers in the high priority stage
may be deferred on the receiving CPU until the out-of-band activity
has quiesced on that CPU. Eventually, the preempted in-band code can
resume normally, which may involve handling the deferred interrupts.

In other words, interrupts are flowing down from the out-of-band to
the in-band interrupt stages, which form a two-stage pipeline for
prioritizing interrupt delivery. The runtime context of the
out-of-band interrupt handlers is known as the *oob stage* of the
pipeline, as opposed to the in-band kernel activities sitting on the
*in-band stage*:

![Alt text](/images/pipeline.png?classes=border,shadow "Interrupt pipeline")

An autonomous core can base its own activities on the oob stage,
interposing on specific IRQ events, for delivering real-time
capabilities to a particular set of applications. Meanwhile, the main
kernel operations keep going over the in-band stage unaffected, only
delayed by short preemption times for running the out-of-band work.
