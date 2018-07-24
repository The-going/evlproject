---
title: "Atomic operations"
menuTitle: "Atomic operations"
date: 2018-06-27T17:17:25+02:00
weight: 5
draft: false
---

The effect of [virtualizing interrupt protection]({{%relref
"pipeline/porting/arch.md" %}}) must be reversed for atomic helpers in
*asm-generic/atomic.h*, *asm-generic/bitops/atomic.h* and
*asm-generic/cmpxchg-local.h*, so that no interrupt can preempt their
execution, regardless of the stage their caller live on.

This is required to keep those helpers usable on data which might be
accessed concurrently from both stages.

The usual way to revert such virtualization consists of delimiting the
protected section with `hard_local_irq_save()`,
`hard_local_irq_restore()` calls, in replacement for
`local_irq_save()`, `local_irq_restore()` respectively.
