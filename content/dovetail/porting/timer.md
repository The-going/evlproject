---
title: "Tick devices"
date: 2018-06-27T17:15:23+02:00
weight: 93
---

## Proxy tick device

The proxy tick device is a synthetic clock event device for handing
over the control of the hardware tick device to a high-precision,
out-of-band timing logic, [which cannot be delayed]({{%relref
"/_index.md#dual-kernel-upsides" %}}) by the in-band kernel code.
With this proxy in place, any out-of-band code can gain control over
the timer hardware for carrying out its own timing duties. In the same
move, it is required to honor the timing requests received from the
in-band timer layer (i.e. hrtimers) since the latter won't be able to
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

> Adapting the ARM Global timer driver to out-of-band timing

```
--- a/drivers/clocksource/arm_global_timer.c
+++ b/drivers/clocksource/arm_global_timer.c
@@ -156,11 +156,11 @@ static irqreturn_t gt_clockevent_interrupt(int irq, void *dev_id)
 	 *	the Global Timer flag _after_ having incremented
 	 *	the Comparator register	value to a higher value.
 	 */
-	if (clockevent_state_oneshot(evt))
+	if (clockevent_is_oob(evt) || clockevent_state_oneshot(evt))
 		gt_compare_set(ULONG_MAX, 0);
 
 	writel_relaxed(GT_INT_STATUS_EVENT_FLAG, gt_base + GT_INT_STATUS);
-	evt->event_handler(evt);
+	clockevents_handle_event(evt);
 
 	return IRQ_HANDLED;
 }
@@ -171,7 +171,7 @@ static int gt_starting_cpu(unsigned int cpu)
 
 	clk->name = "arm_global_timer";
 	clk->features = CLOCK_EVT_FEAT_PERIODIC | CLOCK_EVT_FEAT_ONESHOT |
-		CLOCK_EVT_FEAT_PERCPU;
+		CLOCK_EVT_FEAT_PERCPU | CLOCK_EVT_FEAT_PIPELINE;
 	clk->set_state_shutdown = gt_clockevent_shutdown;
 	clk->set_state_periodic = gt_clockevent_set_periodic;
 	clk->set_state_oneshot = gt_clockevent_shutdown;
@@ -195,11 +195,6 @@ static int gt_dying_cpu(unsigned int cpu)
 	return 0;
 }
 
@@ -302,8 +307,8 @@ static int __init global_timer_of_register(struct device_node *np)
 		goto out_clk;
 	}
 
-	err = request_percpu_irq(gt_ppi, gt_clockevent_interrupt,
-				 "gt", gt_evt);
+	err = __request_percpu_irq(gt_ppi, gt_clockevent_interrupt,
+				   IRQF_TIMER, "gt", gt_evt);
 	if (err) {
 		pr_warn("global-timer: can't register interrupt %d (%d)\n",
 			gt_ppi, err);
```

This is another example of adapting an existing clock chip driver for
serving out-of-band timing requests, with a subtle change in the way
we should test for the current state of the clock device in the
interrupt handler:

- A real/original device (such as the ARM global timer in this
  example) is switched to _detached_ mode when the proxy tick driver
  hands it over to the autonomous core. Therefore, the ARM global
  timer state is always _detached_ from the standpoint of the kernel
  when proxied, never _oneshot_. For this reason,
  `clockevent_state_oneshot()` would always lead to _false_ in this
  case.

- However, since a real device controlled by the proxy for receiving
  out-of-band events has to be driven in one-shot mode under the hood,
  testing for `clockevent_state_oob()` in addition to
  `clockevent_state_oneshot()` guarantees that we do take the branch,
  setting the comparator register to ULONG_MAX when proxied too.

{{% notice warning %}}
Failing to fix up the way we test for the clock device state would
certainly lead to an interrupt storm with any ARM global timer
suffering erratum 740657, quickly locking up the board.
{{% /notice %}}

### Theory of operations {#proxy-tick-logic}

Calling `tick_install_proxy()` creates an instance of the proxy tick
device on each CPU mentioned in the `cpumask` it receives.  This
routine is also passed the address of a routine which should return a
pre-initialized `struct clock_proxy_device` descriptor for the current
CPU. This routine is called indirectly by `tick_install_proxy()`, for
each CPU marked in `cpumask`.

> Initializing the proxy descriptor

`tick_install_proxy()` first `get_percpu_device()`, allowing the user
to allocate a new synthetic device for the current CPU, then doing the
prep work like chosing some of its settings. The descriptor is defined
as follows:

```
struct clock_proxy_device {
	struct clock_event_device proxy_device;
	struct clock_event_device *real_device;
	void (*handle_inband_event)(struct clock_event_device *dev);
	void (*handle_oob_event)(struct clock_event_device *dev);
};
```

Setting the `.handle_oob_event` handler is required: this is the
address of the routine which should be called each time an out-of-band
tick is received from the underlying timer hardware the proxy
controls.

Handlers from the `.proxy_device` field representing the proxy device
into the clock event framework may be set to valid (non-NULL) values
too, if the user code wants to interpose on the corresponding
events. A default value is set by the `tick_install_proxy()` for each
of them if left NULL by `get_percpu_device()`.

If the user code proxying the tick device prefers dealing with
nanoseconds instead of clock ticks directly, CLOCK_EVT_FEAT_KTIME
should be present in `proxy_device.features` as returned by
`get_percpu_device()`.

`real_device` is a pointer to the actual clock device which the proxy
is about to control on the current CPU, which comes in handy if you
want to pull some information from this device to configure the proxy.

`tick_install_proxy()` can configure the proxy with sensible default
values in `.proxy_device`, but requires a valid out-of-band handler to
be set for `.handle_oob_event()`.

> A `get_percpu_device()` routine preparing a proxy device

```
static DEFINE_PER_CPU(struct clock_proxy_device, tick_device);

static void oob_event_handler(struct clock_event_device *dev)
{
	/*
	 * We are running on the out-of-band stage, in NMI-like mode.
	 * Schedule a tick on the proxy device to satisfy the
	 * corresponding timing request asap.
	 */
	tick_notify_proxy();
}

static struct clock_proxy_device *get_percpu_device(void)
{
	struct clock_proxy_device *dev = raw_cpu_ptr(&tick_device);

	/* If called multiple times, we need to zero the structure. */
	memset(dev, 0, sizeof(*dev));

	/*
	 * Create a proxy which acts as a transparent device, simply
	 * relaying the timing requests to the inband code, without
	 * any additional out-of-band processing.
	 */
	dev->handle_oob_event = oob_event_handler;

	return dev;
}
```

{{% notice tip %}}
The `set_next_event()` or `set_next_ktime()` member can be set in
`dev->proxy_device` with the address of a handler which receives timer
requests from the in-band kernel. This handler is normally implemented
by the autonomous core which takes control over the timer hardware via
the proxy device. Whenever that core determines that a tick is due for
an outstanding request received from such handler, it should call
`tick_notify_proxy()` to signal the event to the main kernel.
{{% /notice %}}

Once the user-supplied `get_percpu_device()` routine returns, the
following events happen in sequence:

1. the proxy device is registered, substituting for the real device.

2. the real device is detached from the `clockevent` layer, switched
   to the CLOCK_EVT_STATE_RESERVED state, which makes it non-eligible
   for any regular operation from the clock event framework. However,
   it is left in a functional state.

3. the proxy device starts controlling the real device under the hood
   to carry out timing requests from the in-band kernel. When the
   `hrtimer` layer from the in-band kernel wants to program the next
   shot of the current tick device, it invokes the `set_next_event()`
   handler of the proxy device, which was defined by the caller (which
   defaults to the real device's `set_next_event()` handler). This
   handler should be scheduling in-band ticks at the requested time
   based on its own timer management.

4. the timer interrupt triggered by the real device is switched to
   out-of-band handling. As a result, `handle_oob_event()` receives
   tick events sent by the real device hardware directly from the
   out-of-band stage of the interrupt pipeline. This ensures
   high-precision timing. From that point, the out-of-band code can
   carry out its own timing duties, in addition to honoring the
   in-band kernel requests for timing.

Step 3. involves emulating ticks scheduled by the in-band kernel by a
software logic controlled by some out-of-band timer management, paced
by the real ticks received as described in step 4. When this logic
decides than the next in-band tick is due, it should call
`tick_notify_proxy()` to trigger the corresponding event for the
in-band kernel, which would honor the pending (hr)timer request.

> Under the hood

```
	     [inband timing request]
	         proxy_dev->set_next_event(proxy_dev)
	             oob_program_event(proxy_dev)
	                 real_dev->set_next_event(real_dev)
	         ...
	         <tick event>
	         handle_inband_event() [out-of-band stage]
	             clockevents_handle_event(real_dev)
	                 handle_oob_event(proxy_dev)
	                     ...(inband tick emulation)...
	                          tick_notify_proxy()
	         ...
	         proxy_irq_handler(proxy_dev) [in-band stage]
	             clockevents_handle_event(proxy_dev)
	                 handle_inband_event(proxy_dev)
```
