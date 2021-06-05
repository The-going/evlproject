---
title: "SPI"
weight: 2
---

EVL provides support for running high frequency SPI transfers which
are useful in implementing closed-loop control systems. Applications
manage the out-of-band transfers from user space via requests sent to
the [SPIDEV
driver](https://git.xenomai.org/xenomai4/linux-evl/-/blob/37f57d73123c3b05b9b4f11d5cd3aa2768010dee/Documentation/spi/spidev.rst),
which exports a user-space API to reach the SPI devices overs a given
bus. To this end, EVL makes a few of strong assumptions:

- a DMA is available for transferring the data over the SPI bus
  between the controller and the device. Since configuring the DMA
  cannot be done from the out-of-band stage, selecting the SPI device
  to talk to over the bus involves stopping the real-time operations,
  in order to switch to the in-band execution stage. Fortunately, most
  use cases which require high frequency, ultra-low latency transfers
  involve talking to a single device over a dedicated SPI bus. In this
  case, the device selection and DMA configuration need to be done
  only once, when setting up the communication.

- the data to be exchanged with the SPI device is stored into a
  fixed-size memory buffer, shared between the kernel and the
  application. The buffer is split in two parts: an input area
  containing the data received from the remote device during the last
  transfer cycle, and an output area containing the data to be sent to
  that device during the same cycle. The shared memory is coherent,
  CPU cache is disabled. The DMA is managed in [pulsed out-of-band
  mode]({{< relref "core/oob-drivers/dma#dma-pulsed-mode" >}}) from
  the
  [SPIDEV](https://git.xenomai.org/xenomai4/linux-evl/-/blob/37f57d73123c3b05b9b4f11d5cd3aa2768010dee/drivers/spi/spidev.c#L439)
  interface.

- while operating in out-of-band mode, the SPI bus cannot be used for
  regular in-band traffic.

- only the SPI _master_ mode is supported for out-of-band operations.

![Alt text](/images/spi-basics.png "Out-of-band capable SPI")

Enabling the out-of-band SPI capabilities is threefold, this requires:

- adding the out-of-band management logic to the [generic SPI master
  framework](https://git.xenomai.org/xenomai4/linux-evl/-/blob/37f57d73123c3b05b9b4f11d5cd3aa2768010dee/drivers/spi/spi.c#L4058). This
  is done once and readily available from EVL.

- having the [SPIDEV
  driver](https://git.xenomai.org/xenomai4/linux-evl/-/blob/37f57d73123c3b05b9b4f11d5cd3aa2768010dee/drivers/spi/spidev.c#L451)
  export the out-of-band I/O interface to applications, which is also
  done once and readily available from EVL.

- adding the needed bits to the SPI controller driver which manages
  the particular controller chip to be used for out-of-band I/O. This
  is the part you may have to implement for your own SPI controller of
  choice. The list of SPI controllers EVL currently extends with
  out-of-band capabilities is visible [at this location]({{< relref
  "core/oob-drivers/_index.md#oob-driver-list" >}}).

### Adding out-of-band capabilities to a SPI controller

As stated in the [introduction]({{< relref
"core/oob-drivers/_index.md" >}}), the out-of-band I/O logic slips
into the regular Linux device driver model as much as possible,
without imposing a separate driver stack. In the SPI case, this means
extending the generic SPI framework with a small set of operations
which control the bus from the [out-of-band]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}) execution
stage. The handlers for those operations must be added to the `struct
spi_controller` descriptor as follows:

- `.prepare_oob_transfer` should finish the setup for preparing a
  particular SPI bus for out-of-band transfers. This is called once,
  after the generic SPI framework has created the shared memory area,
  and configured the DMA. This is the right place to provide for any
  setup which is specific to out-of-band I/O, or any additional sanity
  check which is specific to the controller in such a context. For
  instance, the controller could make sure the size of the data frame
  to send/receive at each transfer is within the bounds supported by
  the hardware. This is an in-band operation, the SPI bus is locked
  for out-of-band traffic which means that regular in-band request to
  the same bus will have to wait until it leaves the out-of-band mode.
  This handler runs upon request from the application to enable
  out-of-band mode (`SPI_IOC_ENABLE_OOB_MODE`) for a given SPI bus.

> Example: the prepare_oob_transfer() handler for the BCM2835 chip
```
static int bcm2835_spi_prepare_oob_transfer(struct spi_controller *ctlr,
					struct spi_oob_transfer *xfer)
{
	/*
	 * The size of a transfer is limited by DLEN which is 16-bit
	 * wide, and we don't want to scatter transfers in out-of-band
	 * mode, so cap the frame size accordingly.
	 */
	if (xfer->setup.frame_len > 65532)
		return -EINVAL;

	return 0;
}
```

- `.start_oob_transfer` should select the SPI device to talk to, set
  the SPI communication settings in the controller (e.g. speed, word
  size), then turn on the DMA operations on the configured channels
  Since the DMA is set in [pulsed mode]({{< relref
  "core/oob-drivers/dma#dma-pulsed-mode" >}}), no transfer takes place
  yet, but once `start_oob_transfer()` has returned, everything should
  be ready to trigger them simply by pulsing the DMA.  Like
  `prepare_oob_transfer()`, this is an in-band operation which is
  invoked by the SPIDEV driver, upon request from the application
  (`SPI_IOC_ENABLE_OOB_MODE`). The bus is first prepared for
  out-of-band operations, before these are effectively started.

> Example: the start_oob_transfer() handler for the BCM2835 chip
```
static void bcm2835_spi_start_oob_transfer(struct spi_controller *ctlr,
					struct spi_oob_transfer *xfer)
{
	struct bcm2835_spi *bs = spi_controller_get_devdata(ctlr);
	struct spi_device *spi = xfer->spi;
	u32 cs = bs->prepare_cs[spi->chip_select], effective_speed_hz;
	unsigned long cdiv;

	/* See bcm2835_spi_prepare_message(). */
	bcm2835_wr(bs, BCM2835_SPI_CS, cs);

	cdiv = bcm2835_get_clkdiv(bs, xfer->setup.speed_hz, &effective_speed_hz);
	xfer->effective_speed_hz = effective_speed_hz;
	bcm2835_wr(bs, BCM2835_SPI_CLK, cdiv);
	bcm2835_wr(bs, BCM2835_SPI_DLEN, xfer->setup.frame_len);

	if (spi->mode & SPI_3WIRE)
		cs |= BCM2835_SPI_CS_REN;
	bcm2835_wr(bs, BCM2835_SPI_CS,
		   cs | BCM2835_SPI_CS_TA | BCM2835_SPI_CS_DMAEN);
}
```

- `.pulse_oob_transfer` is called when the application asks for
  triggering the next transfer by a request to the SPIDEV driver
  (`SPI_IOC_RUN_OOB_XFER`). This handler is called to apply the
  controller-specific tweaks which might be needed before the generic
  SPI framework pulses the DMA, causing the I/O operation to take
  place. This is an out-of-band operation; what this handler may do is
  restricted to the set of calls available to the out-of-band
  execution stage (reading/writing a couple of I/O registers should be
  enough in most cases here).

> Example: the pulse_oob_transfer() handler for the BCM2835 chip
```
static void bcm2835_spi_pulse_oob_transfer(struct spi_controller *ctlr,
					struct spi_oob_transfer *xfer)
{
	struct bcm2835_spi *bs = spi_controller_get_devdata(ctlr);

	/* Reload DLEN for the next pulse. */
	bcm2835_wr(bs, BCM2835_SPI_DLEN, xfer->setup.frame_len);
}
```

- `.terminate_oob_transfer` should stop and disable out-of-band DMA
  operations for the controller. This handler is called when the
  generic SPI framework was told by the SPIDEV driver to leave the
  out-of-band management mode for the controller
  (`SPI_IOC_DISABLE_OOB_MODE`). This is an in-band operation. Once the
  out-of-band mode is left, the bus is available for regular in-band
  traffic anew.

> Example: the terminate_oob_transfer() handler for the BCM2835 chip
```
static void bcm2835_spi_reset_hw(struct bcm2835_spi *bs)
{
	u32 cs = bcm2835_rd(bs, BCM2835_SPI_CS);

	/* Disable SPI interrupts and transfer */
	cs &= ~(BCM2835_SPI_CS_INTR |
		BCM2835_SPI_CS_INTD |
		BCM2835_SPI_CS_DMAEN |
		BCM2835_SPI_CS_TA);
	/*
	 * Transmission sometimes breaks unless the DONE bit is written at the
	 * end of every transfer.  The spec says it's a RO bit.  Either the
	 * spec is wrong and the bit is actually of type RW1C, or it's a
	 * hardware erratum.
	 */
	cs |= BCM2835_SPI_CS_DONE;
	/* and reset RX/TX FIFOS */
	cs |= BCM2835_SPI_CS_CLEAR_RX | BCM2835_SPI_CS_CLEAR_TX;

	/* and reset the SPI_HW */
	bcm2835_wr(bs, BCM2835_SPI_CS, cs);
	/* as well as DLEN */
	bcm2835_wr(bs, BCM2835_SPI_DLEN, 0);
}

static void bcm2835_spi_terminate_oob_transfer(struct spi_controller *ctlr,
					struct spi_oob_transfer *xfer)
{
	struct bcm2835_spi *bs = spi_controller_get_devdata(ctlr);

	bcm2835_spi_reset_hw(bs);
}

```

Eventually, the SPI controller should fill the corresponding function
pointers into its descriptor with the address of the out-of-band
handlers.

> Example: declaring the out-of-band handlers for the BCM2835 chip
```
static int bcm2835_spi_probe(struct platform_device *pdev)
{
	struct spi_controller *ctlr;
	struct bcm2835_spi *bs;
	int err;

	ctlr = devm_spi_alloc_master(&pdev->dev, ALIGN(sizeof(*bs),
						  dma_get_cache_alignment()));
	if (!ctlr)
		return -ENOMEM;

	platform_set_drvdata(pdev, ctlr);

	...
	ctlr->prepare_oob_transfer = bcm2835_spi_prepare_oob_transfer;
	ctlr->start_oob_transfer = bcm2835_spi_start_oob_transfer;
	ctlr->pulse_oob_transfer = bcm2835_spi_pulse_oob_transfer;
	ctlr->terminate_oob_transfer = bcm2835_spi_terminate_oob_transfer;
	...
}
```

---

{{<lastmodified>}}
