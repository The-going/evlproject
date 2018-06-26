---
title: "Raw printk support"
menuTitle: "Serial debugging"
weight: 97
---

Unless you are lucky enough to have an ICE for debugging hard issues
involving out-of-band contexts, you might have to resort to basic
printk-style debugging over a serial line. Although the `printk()`
machinery can be used from out-of-band context when Dovetail is
enabled, the output is deferred until the in-band stage gets back in
control, which means that:

- you can't reliably trace out-of-band code on the spot, deferred
  output issued from an out-of-band context, or from a section of code
  running with interrupts disabled in the CPU may appear after
  subsequent in-band messages under some circumstances, due to a
  buffering effect.

- if the debug traces are sent at high pace (e.g. from an out-of-band
  IRQ handler every few hundreds of microseconds), the machine is
  likely to come to a stall due to the massive output the heavy
  `printk()` machinery would have to handle, leading to an apparent
  lockup.

The only sane option for printk-like debugging in demanding
out-of-band context is using the `raw_printk()` routine for issuing
raw debug messages to a serial console, so that you may get some
sensible feedback for understanding what is going on with the
execution flow. This feature should be enabled by turning on
`CONFIG_RAW_PRINTK`, otherwise all output sent to `raw_printk()` is
discarded.

Because a stock serial console driver won't be usable from out-of-band
context, enabling raw printk support requires adapting the serial
console driver your platform is using, by adding a raw write handler
to the console description. Just like the `write()` handler, the
`write_raw()` output handler receives a console pointer, the character
string to output and its length as parameters. This handler should
send the characters to the UART as quickly as possible, with little to
no preparation.

All output formatted by the generic `raw_printk()` routine is passed
to the raw write handler of the current serial console driver if
present. Calls to the raw output handler are serialized in
`raw_printk()` by holding a [hard spinlock]({{%relref
"dovetail/pipeline/locking.md#new-spinlocks" %}}), which means
that interrupts are disabled in the CPU when running the handler.

A raw write handler is normally derived from the regular write handler
for the same serial console device, skipping any in-band locking
construct, only waiting for the bare minimum time for the output to
drain in the UART since we want to keep interrupt latency low.

{{% notice warning %}}
You cannot expect mixed output sent via `printk()` then `raw_printk()`
to appear in the same sequence as their respective calls: normal
`printk()` output may be deferred for an undefined amount of time
until some console driver sends it to the terminal device, which may
involve a task rescheduling. On the other hand, `raw_printk()`
immediately writes the output to the hardware device, bypassing any
buffering from `printk()`. So the output from a sequence of `printk()`
followed by `raw_printk()` may appear in the opposite order on the
terminal device. The converse never happen though.
{{% /notice %}}

> Adding RAW_PRINTK support to the AMBA PL011 serial driver

```
--- a/drivers/tty/serial/amba-pl011.c
+++ b/drivers/tty/serial/amba-pl011.c
@@ -2206,6 +2206,40 @@ static void pl011_console_putchar(struct uart_port *port, int ch)
 	pl011_write(ch, uap, REG_DR);
 }
 
+#ifdef CONFIG_RAW_PRINTK
+
+/*
+ * The uart clk stays on all along in the current implementation,
+ * despite what pl011_console_write() suggests, so for the time being,
+ * just emit the characters assuming the chip is clocked. If the clock
+ * ends up being turned off after writing, we may need to clk_enable()
+ * it at console setup, relying on the non-zero enable_count for
+ * keeping pl011_console_write() from disabling it.
+ */
+static void
+pl011_console_write_raw(struct console *co, const char *s, unsigned int count)
+{
+	struct uart_amba_port *uap = amba_ports[co->index];
+	unsigned int old_cr, new_cr, status;
+
+	old_cr = readw(uap->port.membase + UART011_CR);
+	new_cr = old_cr & ~UART011_CR_CTSEN;
+	new_cr |= UART01x_CR_UARTEN | UART011_CR_TXE;
+	writew(new_cr, uap->port.membase + UART011_CR);
+
+	while (count-- > 0) {
+		if (*s == '\n')
+			pl011_console_putchar(&uap->port, '\r');
+		pl011_console_putchar(&uap->port, *s++);
+	}
+	do
+		status = readw(uap->port.membase + UART01x_FR);
+	while (status & UART01x_FR_BUSY);
+	writew(old_cr, uap->port.membase + UART011_CR);
+}
+
+#endif  /* !CONFIG_RAW_PRINTK */
+
 static void
 pl011_console_write(struct console *co, const char *s, unsigned int count)
 {
@@ -2406,6 +2440,9 @@ static struct console amba_console = {
 	.device		= uart_console_device,
 	.setup		= pl011_console_setup,
 	.match		= pl011_console_match,
+#ifdef CONFIG_RAW_PRINTK
+	.write_raw	= pl011_console_write_raw,
+#endif
 	.flags		= CON_PRINTBUFFER | CON_ANYTIME,
 	.index		= -1,
 	.data		= &amba_reg,
```

### ARM-specific raw console driver

The vanilla ARM kernel port already provides an UART-based raw output
routine called `printascii()` when `CONFIG_DEBUG_LL` is enabled,
provided the right debug UART channel is defined too
(`CONFIG_DEBUG_UART_xx`).

When `CONFIG_RAW_PRINTK` and `CONFIG_DEBUG_LL` are both defined in the
kernel configuration, the ARM implementation of Dovetail automatically
registers a special console device for emitting debug output (see
_arch/arm/kernel/raw\_printk.c_), which redirects calls to its raw
write handler by `raw_printk()` to `printascii()`. In other words, if
`CONFIG_DEBUG_LL` already provides you with a functional debug output
channel, you don't need the active serial console driver to implement
a raw write handler for enabling `raw_printk()`, the raw console
device should handle `raw_printk()` requests just fine.

{{% notice warning %}}
Enabling `CONFIG_DEBUG_LL` with a wrong UART debug channel is a common
cause of lockup at boot. You do want to make sure that the proper
`CONFIG_DEBUG_UART_xx` symbol matching your hardware is selected along
with `CONFIG_DEBUG_LL`.
{{% /notice %}}

---

{{<lastmodified>}}
