---
title: "Locking"
date: 2018-06-27T17:23:53+02:00
weight: 7
draft: false
---

## Additional spinlock types {#new-spinlocks}

The pipeline core introduces two spinlock types:

+ *hard* spinlocks manipulate the CPU interrupt mask, and don't affect
  the kernel preemption state in locking/unlocking operations.

This type of spinlock is useful for implementing a critical section to
serialize concurrent accesses from both in-band and out-of-band
contexts, i.e. from in-band and oob stages. Obviously, sleeping into a
critical section protected by a hard spinlock would be a very bad
idea. In other words, hard spinlocks are not subject to virtual
interrupt masking, therefore can be used to serialize with out-of-band
activities, including from the in-band kernel code. At any rate, those
sections ought to be quite short, for keeping latency low.

+ *mutable* spinlocks are used internally by the pipeline core to
  protect access to IRQ descriptors (`struct irq_desc::lock`), so that
  we can keep the original locking scheme of the generic IRQ core
  unmodified for handling out-of-band interrupts.

Mutable spinlocks behave like *hard* spinlocks when traversed by the
low-level IRQ handling code on entry to the pipeline, or common *raw*
spinlocks otherwise, preserving the kernel (virtualized) interrupt and
preemption states as perceived by the in-band context. This type of
lock is not meant to be used in any other situation.

## Lockdep support

The lock validator automatically reconciles the real and virtual
interrupt states, so it can deliver proper diagnosis for locking
constructs defined in both in-band and out-of-band contexts. This
means that *hard* and *mutable* spinlocks are included in the
validation set when LOCKDEP is enabled.

{{% notice warning %}}
These two additional types are subject to LOCKDEP analysis. However,
be aware that latency figures are likely to be **really bad** when
LOCKDEP is enabled, due to the large amount of work the lock validator
may have to do with interrupts disabled for the CPU (i.e. _hard_
locking) for enforcing critical sections.
{{% /notice %}}
