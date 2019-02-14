---
title: "Interrupt Protection"
date: 2018-07-01T17:54:11+02:00
weight: 6
draft: false
---

## Disabling interrupts in the CPU {#hard-irq-protection}

The `local_irq_save()` and `local_irq_disable()` helpers are no more
disabling interrupts in the CPU when interrupt pipelining is enabled,
but only disable interrupt events [virtually for the in-band
stage]({{%relref "pipeline/optimistic.md#virtual-i-flag" %}}).

A set of helpers is provided for manipulating the interrupt disable
flag in the CPU instead. When CONFIG_IRQ_PIPELINE is disabled, this
set maps 1:1 over the regular `local_irq_*()` API.

|     Original/Virtual        |       Non-virtualized call         |
| :-------------------------- |:---------------------------------- |
|  local_save_flags(flags)    |   flags = hard_local_save_flags()  |
|  local_irq_disable()	      |   hard_local_irq_disable()         |
|  local_irq_enable()	      |   hard_local_irq_enable()          |
|  local_irq_save(flags)      |   flags = hard_local_irq_save()    |
|  local_irq_restore(flags)   |   hard_local_irq_restore(flags)    |
|  irqs_disabled()            |   hard_irqs_disabled()             |
|  irqs_disabled_flags(flags) |   hard_irqs_disabled_flags(flags)  |

## Stalling the oob stage {# head-stall-flag}

Just like the in-band stage is affected by the state of the [virtual
interrupt disable flag]({{%relref
"pipeline/optimistic.md#virtual-i-flag" %}}), the interrupt state of
the oob stage is controlled by a dedicated _stall bit_ flag in the
oob stage's status. In combination with the interrupt disable bit in
the CPU, this software bit controls interrupt delivery to the oob
stage.

When this _stall bit_ is set, interrupts which might be pending in the
oob stage's event log of the current CPU are not played. Conversely,
the out-of-band handlers attached to pending IRQs are fired when the
_stall bit_ is clear. The following table represents the equivalent
calls affecting the stall bit for each stage:

|     Root stage operation    |       Head stage operation         |
| :-------------------------- |:---------------------------------- |
|  local_save_flags(flags)    |             -none-                 |
|  local_irq_disable()	      |        oob_irq_disable()           |
|  local_irq_enable()	      |        oob_irq_enable()            |
|  local_irq_save(flags)      |    flags = oob_irq_save()          |
|  local_irq_restore(flags)   |    oob_irq_restore(flags)          |
|  irqs_disabled()            |    oob_irqs_disabled()             |
|  irqs_disabled_flags(flags) |             -none-                 |

{{% notice tip %}}
Using this set of helpers only makes sense from out-of-band code,
typically in the real-time core implementation. Calling them from any
other context would be highly suspicious. Adapting in-band code to
interrupt pipelining might involve using the `hard_local_*()`
set instead, but only in marginal cases, always with extreme care.
{{% /notice %}}
