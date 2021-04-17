---
title: "DMA"
weight: 1
---

A cornerstone of many real-time capable drivers which can process
requests from the out-of-band stage is DMA support. Being able to
offload data transfers to a DMA unit goes a long way toward
implementing efficient acquisition loops, especially if they have to
run at high frequency, sparing precious CPU cycles.

In order to support out-of-band transactions, a DMA controller
(e.g. `bcm2835-dma`, or `imx-sdma`) must include Dovetail-specific
changes. Since DMA drivers are commonly based on the virtual channel
layer (aka `virt-dma`), Dovetail adds the required changes to this
layer in order to cope with execution from the out-of-band stage. The
current list of DMA controllers which support out-of-band transactions
is available [at this location]({{< relref
"core/oob-drivers/_index.md#oob-driver-list" >}}).

## Out-of-band DMA channel

Dovetail introduces a simple interface for switching a DMA channel to
out-of-band mode, based on special flags passed to any of the
supported prep calls, such as `dmaengine_prep_slave_sg()`,
`dmaengine_prep_slave_single()` or `dmaengine_prep_dma_cyclic()`. This
mode remains active until `dmaengine_terminate_async()` (or the
deprecated `dmaengine_terminate_all()`) routine is called for the
channel, which is therefore reserved for out-of-band operations until
terminated.

Two modes of operation are available for running a DMA channel
out-of-band:

- the common cyclic mode, where an I/O peripheral is driving the data
  transaction, triggering transfers repeatedly on a periodic basis. If
  the `DMA_OOB_INTERRUPT` flag is passed to
  `dmaengine_prep_dma_cyclic()`, the DMA completion callback set for
  the TX descriptor (see `struct dma_async_tx_descriptor`) is called
  at the end of each cycle from the out-of-band stage, so that such
  callback can wake up real-time capable tasks without involving any
  non-deterministic in-band activity. A typical use case is that an
  application on the Linux side with strict low latency timing
  requirement runs the data exchanged with an audio codec through some
  processing pipeline.

- the new pulsed mode introduced by Dovetail, in which the kernel
  software wants to trigger every data transfer manually and
  repeatedly, acting as the master side of the transaction. This mode
  should be available from any type of DMA transaction the out-of-band
  capable driver supports, but the cyclic one. It is turned on by
  passing the `DMA_OOB_PULSE` flag to the corresponding prep call,
  such as `dmaengine_prep_slave_sg()` or
  `dmaengine_prep_slave_single()`. Once the transfer is completed, any
  callback set for the TX descriptor is fired from the out-of-band
  stage. The key aspect is the ability to trigger multiple transfers
  and receive the corresponding completion events directly from the
  out-of-band stage using a single TX descriptor, which is different
  from the usual case in which every TX descriptor describes a single
  transfer which might be delayed by other pending operations.  A
  typical use case is with implementing closed-control loops driven by
  the Linux system with stringent timing requirements, for instance
  over a [SPI bus]({{< relref "core/oob-drivers/spi.md" >}}).

## Programming a DMA channel for out-of-band operation

### Cyclic transaction

The way to set up a cyclic transaction with out-of-band completion
events is straightforward:

1. create a cyclic DMA transaction via `dmaengine_prep_dma_cyclic()`,
OR'ing the `DMA_OOB_INTERRUPT` flag into the _flags_ parameter. This
flag ensures that the completion callback set into the DMA TX
descriptor is called from the out-of-band stage.

2. submit the transaction by a call to `dmaengine_submit()`,

3. finally bring this transaction to active state by issuing the
pending DMA requests the way you would do for common transactions
using `dma_async_issue_pending()`. Once your out-of-band transaction
is picked by the DMA engine for execution, it has a lock on the
channel until terminated.

> Setting up a DMA transaction for cyclic out-of-band transfers

![Alt text](/images/oob-dma-cyclic.png?featherlight=false "Cyclic out-of-band transaction")

### Pulsed mode transaction {#dma-pulsed-mode}

A DMA transaction of pulsed transfers is a common slave transaction
for the most part, except that each transfer must be started manually
by a call to [dma_pulse_oob()]({{< relref "#dma_pulse_oob" >}}) for
the related channel. To set it up, you need to:

1. create a slave DMA transaction via `dmaengine_prep_slave_sg()` or
`dmaengine_prep_slave_single()` for instance, OR'ing the
`DMA_OOB_PULSE` flag into the _flags_ parameter. This flag ensures
that:
	- each transfer in either direction can only be started at your
	command by a call to [dma_pulse_oob()]({{< relref "#dma_pulse_oob"
	>}}) instead of leaving this decision to the DMA engine,

	- the completion callback set into the DMA TX descriptor is called from the
	out-of-band stage each time such a transfer has completed.

2. submit the transaction by a call to `dmaengine_submit()`,

3. finally bring this transaction to active state by issuing the
pending DMA requests the way you would do for common transactions
using `dma_async_issue_pending()`. Once your out-of-band transaction
is picked by the DMA engine for execution, it has a lock on the
channel until terminated.

> Setting up a DMA transaction for pulsed out-of-band transfers

![Alt text](/images/oob-dma-pulsed.png?featherlight=false "Pulsed out-of-band transaction")

---

{{< proto dma_pulse_oob >}}
int dma_pulse_oob(struct dma_chan *chan)
{{< /proto >}}

This routine triggers the next transfer on a DMA channel set up for a
pulsed mode transaction, either for sending or receiving data
depending on the prep call.

{{% argument chan %}}
The DMA channel descriptor for which to start the next transfer.
{{% /argument %}}

Zero is returned on success, otherwise:

- -EIO. No transfer from an out-of-band transaction is ready to be
  started manually. Typically, _chan_ was not previously switched to
  out-of-band operation mode by submitting a TX descriptor bearing
  `DMA_OOB_PULSE`, followed by a call to `dmaengine_submit()`. You may
  also need to tell the DMA engine to start processing the pending
  transactions by calling `dma_async_issue_pending()` right after
  submitting the TX descriptor.

---

{{% notice info %}}
An out-of-band completion handler executes in out-of-band IRQ context,
so you may only run code which is legit there.
{{% /notice %}}

{{<lastmodified>}}
