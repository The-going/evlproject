---
title: "Timer management"
date: 2018-06-27T17:15:23+02:00
weight: 5
draft: false
---

## Proxy tick device

The proxy tick device is a synthetic clock event device for handing
over the control of the hardware tick device to a high-precision,
out-of-band timing logic, [which does not run into delays]({{%relref
"/_index.md#dual-kernel-upsides" %}}) caused by the in-band kernel code.
    
With this proxy in place, any out-of-band code can gain control over
the timer hardware for carrying out its own timing duties. In the same
move, it is required to honor the timing requests received from the
in-band timer core (i.e. hrtimers) since the latter won't be able to
program timer events directly into the hardware while the proxy is
active.

In other words, the proxy tick device shares the functionality of the
actual device between the in-band and out-of-band contexts, with only
the latter actually programming the hardware.

## Changes to the clock chip devices

The proxy device stacks over a real clock chip device, controlling
it. Clock chips which may be controlled by the proxy tick device need
their drivers to be specifically adapted for such use, as follows:

+ `clockevents_handle_event()` must be used to invoke the event
handler from the interrupt handler, instead of dereferencing `struct
clock_event_device::event_handler` directly.

+ `struct clock_event_device::irq` must be properly set to the actual
IRQ number signaling an event from this device.

+ `struct clock_event_device::features` must include
`CLOCK_EVT_FEAT_PIPELINE`.

+ `__IRQF_TIMER` must be set for the action handler of the timer
device interrupt.

> Adapting the ARM architected timer driver to out-of-band timing

```
--- a/drivers/clocksource/arm_arch_timer.c
+++ b/drivers/clocksource/arm_arch_timer.c
@@ -585,7 +585,7 @@ static __always_inline irqreturn_t timer_handler(const int access,
 	if (ctrl & ARCH_TIMER_CTRL_IT_STAT) {
 		ctrl |= ARCH_TIMER_CTRL_IT_MASK;
 		arch_timer_reg_write(access, ARCH_TIMER_REG_CTRL, ctrl, evt);
-		evt->event_handler(evt);
+		clockevents_handle_event(evt);
 		return IRQ_HANDLED;
 	}
 
@@ -704,7 +704,7 @@ static int arch_timer_set_next_event_phys_mem(unsigned long evt,
 static void __arch_timer_setup(unsigned type,
 			       struct clock_event_device *clk)
 {
-	clk->features = CLOCK_EVT_FEAT_ONESHOT;
+	clk->features = CLOCK_EVT_FEAT_ONESHOT | CLOCK_EVT_FEAT_PIPELINE;
 
 	if (type == ARCH_TIMER_TYPE_CP15) {
 		if (arch_timer_c3stop)
```

{{% notice note %}}
Only oneshot-capable clock event devices can be shared via the proxy tick device.
{{% /notice %}}
