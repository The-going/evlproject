---
title: "Timer management"
date: 2018-06-27T17:15:23+02:00
weight: 5
draft: false
---

## Proxy tick device

The proxy tick device is a synthetic clock event device for handing
over the control of the hardware tick device to a high-precision,
out-of-band timing logic, [which cannot be delayed]({{%relref
"/_index.md#dual-kernel-upsides" %}}) by the in-band kernel code.
With this proxy in place, any out-of-band code can gain control over
the timer hardware for carrying out its own timing duties. In the same
move, it is required to honor the timing requests received from the
in-band timer core (i.e. hrtimers) since the latter won't be able to
program timer events directly into the hardware while the proxy is
active.

In other words, the proxy tick device shares the functionality of the
actual device between the in-band and out-of-band contexts, with only
the latter actually programming the hardware.

### Adapting clock chip devices for proxying

The proxy tick device borrows a real clock chip device from the
in-band kernel, controlling it under the hood while substituting for
the current tick device. Clock chips which may be controlled by the
proxy tick device need their drivers to be specifically adapted for
such use, as follows:

+ `clockevents_handle_event()` must be substituted to any open-coded
invocation of the event handler in the interrupt handler.

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

### Theory of operations {#proxy-tick-logic}

Calling `tick_install_proxy()` creates an instance of the proxy tick
device on each CPU mentioned in the `cpumask` it receives.  This
routine is also passed a pointer to a `struct proxy_tick_ops`
operation descriptor (`ops`), defining a few handlers the caller
should provide for contolling the proxy device.

> The proxy operation descriptor

```
struct proxy_tick_ops {
	void (*register_device)(struct clock_event_device *proxy_ced,
				struct clock_event_device *real_ced);
	void (*unregister_device)(struct clock_event_device *proxy_ced,
				  struct clock_event_device *real_ced);
	void (*handle_event)(struct clock_event_device *real_ced);
};
```

`tick_install_proxy()` invokes `ops->register_device()` for doing the
prep work for the synthetic device, allowing the client code to chose
its settings before registering it, typically by a call to
`clockevents_config_and_register()`.

{{% notice note %}}
Although this is not strictly required, it is highly expected that
`register_device()` gives a better rating to the proxy device than the
original tick device's, so that the in-band kernel would substitute the
former for the latter.
{{% /notice %}}

> A `register_device()` handler preparing a proxy device

```
static void proxy_device_register(struct clock_event_device *proxy_ced,
				  struct clock_event_device *real_ced)
{
	/*
	 * We know how to deal with delays expressed as counts of
	 * nanosecs directly for programming events (i.e. no need
	 * to translate into cycles).
	 */
	proxy_ced->features |= CLOCK_EVT_FEAT_KTIME;
	/*
	 * The handler which would receive in-band timing
	 * requests.
	 */
	proxy_ced->set_next_ktime = proxy_set_next_ktime;
	proxy_ced->set_next_event = NULL;
	/*
	 * Make sure we substitute for the real device:
	 * advertise a better rating.
	 */
	proxy_ced->rating = real_ced->rating + 1;
	proxy_ced->min_delta_ns = 1;
	proxy_ced->max_delta_ns = KTIME_MAX;
	proxy_ced->min_delta_ticks = 1;
	proxy_ced->max_delta_ticks = ULONG_MAX;

	/* All set, register now. */
	clockevents_register_device(proxy_ced);
}
```

Once the user-supplied `ops->register_device()` handler returns, the
following events happen in sequence:

1. with a rating most likely set to be higher than the current tick
   device's (i.e. the _real_ device), the proxy device substitutes for
   the former.

2. the real device is detached from the `clockevent` core. However, it
   is left in a functional state. The `set_next_event()` handler of
   this device is redirected to the user-supplied
   `ops->handle_event()` handler. This way, every tick received from
   the real device would be passed on to the latter.

3. the proxy device starts controlling the real device under the hood
   to carry out timing requests from the in-band kernel. When the
   `hrtimer` layer from the in-band kernel wants to program the next
   shot of the current tick device, it invokes the `set_next_event()`
   handler of the proxy device, which was defined by the caller. This
   handler should be scheduling in-band ticks at the requested time
   based on its own timer management.

4. the timer interrupt triggered by the real device is switched to
   out-of-band handling. As a result, `ops->handle_event()` receives
   tick events sent by the real device hardware directly from the head
   stage of the interrupt pipeline, over the out-of-band context. This
   ensures high-precision timing. From that point, the out-of-band
   code can carry out its own timing duties, in addition to honoring
   the in-band kernel requests for timing.

Step 3. involves emulating ticks scheduled by the in-band kernel by a
software logic controlled by some out-of-band timer management, paced
by the real ticks received as described in step 4. When this logic
decides than the next in-band tick is due, it should call
`tick_notify_proxy()` to trigger the corresponding event for the
in-band kernel, which would honor the pending (hr)timer request.

> Under the hood

```
clockevents_program_event() [ROOT STAGE]
   proxy_dev->set_next_event(proxy_dev)
      proxy_set_next_ktime(proxy_dev)
         (out-of-band timing logic)
            real_dev->set_next_event(real_dev)
...
<hardware tick event>
      oob_timer_handler() [HEAD STAGE]
         clockevents_handle_event(real_dev)
            ops->handle_event(proxy_dev)
               (out-of-band timing logic)
	          tick_notify_proxy()  /* schedules hrtimer tick */
...
<synthetic tick event>
      proxy_irq_handler(proxy_dev) [ROOT stage]
         clockevents_handle_event(proxy_dev)
            hrtimer_interrupt(proxy_dev)
```
