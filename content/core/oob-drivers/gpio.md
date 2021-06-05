---
title: "GPIO"
weight: 3
---

Dealing with GPIOs from the [out-of-band]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}) execution stage
enables the application to always respond to external signals within a
few microseconds regardless of the [in-band workload]({{< relref
"core/benchmarks/_index.md#stress-load" >}}) running in parallel on
the system. Enabling [CONFIG_GPIOLIB_OOB]({{< relref
"core/build-steps#core-kconfig" >}}) in the kernel configuration turns
on such such capability in the regular [GPIOLIB
driver](https://git.xenomai.org/xenomai4/linux-evl/-/blob/3451245cdc9846835f8a2786767b17037ee13dda/drivers/gpio/gpiolib-cdev.c#L463),
which depends on the EVL core.  The out-of-band GPIO support is
available to applications using a couple of additional I/O requests to
the character device interface exported by this driver to application
code running in user-space (i.e. based on
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) and
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html) calls for
the most part). This inner interface is tersely documented, but you
may find your way by having a look at this demo code available from
the [mainline kernel
tree](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/gpio).

{{% notice tip %}}
This driver interface is used by the
[libgpiod](https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/)
API internally.
{{% /notice %}}

### Out-of-band GPIO operations in a nutshell

As stated in the [introduction]({{< relref
"core/oob-drivers/_index.md" >}}), the out-of-band I/O logic slips
into the regular Linux device driver model as much as possible,
without imposing a separate driver stack. In the GPIO case, we have
been able to add the out-of-band support to an existing driver such as
the [GPIOLIB
core](https://git.xenomai.org/xenomai4/linux-evl/-/blob/3451245cdc9846835f8a2786767b17037ee13dda/drivers/gpio/gpiolib-cdev.c)
instead of providing a dedicated driver.

This translates as follows:

- Common GPIO handling operations in this driver are extended with the
  specific
  [GPIOHANDLE_REQUEST_OOB](https://git.xenomai.org/xenomai4/linux-evl/-/blob/37f57d73123c3b05b9b4f11d5cd3aa2768010dee/include/uapi/evl/devices/gpio.h#L8)
  flag, which tells the GPIOLIB core about our intent to operate a
  GPIO pin directly from the out-of-band execution stage, for input
  and/or output. Line set up and configuration are still done using
  the regular
  [ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html)
  interface, since these are in-band operations by design.

- Waiting for a GPIO event is done by calling the [oob_read()]({{<
  relref "core/user-api/io/_index.md#oob_read" >}}) I/O service,
  instead of
  [read(2)](http://man7.org/linux/man-pages/man2/read.2.html).

- Toggling a GPIO state is done by calling the [oob_ioctl()]({{< relref
  "core/user-api/io/_index.md#oob_ioctl" >}}) I/O service, instead of
  [ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html).

### Out-of-band line event and line handle requests

In order to use the out-of-band GPIO features, one simply needs to add
the
[GPIOHANDLE_REQUEST_OOB](https://git.xenomai.org/xenomai4/linux-evl/-/blob/37f57d73123c3b05b9b4f11d5cd3aa2768010dee/include/uapi/evl/devices/gpio.h#L8)
flag defined by the EVL core to the common `GPIOHANDLE_REQUEST_INPUT`,
`GPIOHANDLE_REQUEST_OUTPUT` operation flags, when issuing the
`GPIO_GET_LINEEVENT_IOCTL` and `GPIO_GET_LINEHANDLE_IOCTL`
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) requests
respectively. For instance, this is a fragment of code adapted from the
[latmus application]({{< relref
"core/benchmarks/_index.md#latmus-gpio-response-time" >}}) which
measures response time to GPIO events:

```
#include <sys/ioctl.h>
#include <linux/gpio.h>
#include <uapi/evl/devices/gpio.h>

static void setup_gpio_pins(int *fds)
{
	struct gpiohandle_request out;
	struct gpioevent_request in;
	int ret;

	in.handleflags = GPIOHANDLE_REQUEST_INPUT | GPIOHANDLE_REQUEST_OOB;
	in.eventflags = GPIOEVENT_REQUEST_RISING_EDGE;
	in.lineoffset = gpio_inpin; /* Input pin number */
	strcpy(in.consumer_label, "latmon-pulse");

	ret = ioctl(gpio_infd, GPIO_GET_LINEEVENT_IOCTL, &in);
	if (ret)
		error(1, errno, "ioctl(GPIO_GET_LINEEVENT_IOCTL)");

	/* in.fd now contains the oob-capable line event descriptor. */

	out.lineoffsets[0] = gpio_outpin; /* Output pin number */
        out.lines = 1;
	out.flags = GPIOHANDLE_REQUEST_OUTPUT | GPIOHANDLE_REQUEST_OOB;
        out.default_values[0] = 1;
	strcpy(out.consumer_label, "latmon-ack");

	ret = ioctl(gpio_outfd, GPIO_GET_LINEHANDLE_IOCTL, &out);
	if (ret)
		error(1, errno, "ioctl(GPIO_GET_LINEHANDLE_IOCTL)");

	/* out.fd now contains the oob-capable line handle descriptor. */

	fds[0] = in.fd;
	fds[1] = out.fd;
}
```

Once a file descriptor is obtained on the GPIO controller - like
`/dev/gpiochip0` - for input (`gpio_infd`) and output (`gpio_outfd`),
the application may ask for:

- a line event descriptor for receiving GPIO interrupts directly from
  the out-of-band stage, by waiting on the [oob_read()]({{< relref
  "core/user-api/io/_index.md#oob_read" >}}) I/O service.

- a line handle descriptor for changing the state of GPIO pins
  directly from the out-of-band stage, by calling the
  [oob_ioctl()]({{< relref "core/user-api/io/_index.md#oob_ioctl" >}})
  I/O service, with the `GPIOHANDLE_SET_LINE_VALUES_IOCTL` request code.

A thread which responds to GPIO events on an input pin by flipping the
state of an output pin, all from the out-of-band stage could look like
this:

```
static void *gpio_responder_thread(void *arg)
{
	struct gpiohandle_data data = { 0 };
	struct gpioevent_data event;
	const int ackval = 0;	/* Remote observes falling edges. */
	int fds[2], efd, ret;

	setup_gpio_pins(fds);

	/*
	 * Attach to the EVL core so that we may issue out-of-band
	 * requests.
	 */
	efd = evl_attach_self("/gpio-responder:%d", getpid());
	if (efd < 0)
		error(1, -efd, "evl_attach_self() failed");

	/*
	 * Loop: wait for the next event, then trigger an edge by flipping
	 * the pin state.
	 */
	for (;;) {
		data.values[0] = !ackval;
		ret = oob_ioctl(fds[1], GPIOHANDLE_SET_LINE_VALUES_IOCTL, &data);
		if (ret)
			error(1, errno,
			"ioctl(GPIOHANDLE_SET_LINE_VALUES_IOCTL) failed");

		/* Wait for the next interrupt on the input pin. */
		ret = oob_read(fds[0], &event, sizeof(event));
		if (ret != sizeof(event))
			break;

		/* Start flipping the output pin. */
		data.values[0] = ackval;
		ret = oob_ioctl(fds[1], GPIOHANDLE_SET_LINE_VALUES_IOCTL, &data);
		if (ret)
			error(1, errno,
				"ioctl(GPIOHANDLE_SET_LINE_VALUES_IOCTL) failed");
	}

	return NULL;
}
```

---

{{<lastmodified>}}
