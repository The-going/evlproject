---
title: "Real-time I/O drivers"
weight: 9
pre: "&#9702; "
---

The real-time I/O support EVL provides for is based on the ability
some kernel drivers have to handle [out-of-band interrupts]({{< relref
"dovetail/pipeline/_index.md#two-stage-pipeline" >}}) triggered by the
hardware, along with [out-of-band requests]({{< relref
"core/kernel-api/_index.md#evl-driver-definition" >}}) issued by
applications. To achieve this, such driver may depend on the EVL
[kernel API]({{< relref "core/kernel-api/function_index/_index.md"
>}}) which in turn depends on [Dovetail]({{< relref
"dovetail/_index.md" >}}). This way, both the incoming events and
outgoing requests are expedited by the EVL core, without being delayed
by the regular GPOS work which is still handled normally on the
in-band execution stage.

### Integrated vs separated real-time I/O support

Over the years, the usual approach followed by dual kernel systems in
order to provide for real-time I/O support has been to build their own
separate, ad hoc driver stack, re-implementing a marginal portion of
the common driver stack. The arguments invoked for doing so have been
that:

- a strict separation between both code bases would prevent any
  non-deterministic behavior from the common Linux driver stack to
  spill over into the real-time execution, by design.

- such a separate implementation would be required in order to
  optimize for real-time performance.

- the real-time driver stack would be easier to maintain, not being
  affected by conflicting updates to the original kernel code it is
  based on.

The typical way to implement such a driver is to start from a copy of
a mainline kernel driver available for the target device, amending the
sources heavily in order to use the real-time core API instead of the
common kernel API, dropping any code which would not be required for
supporting the real-time applications in the same move. At the end of
the day, such real-time variant of a driver has diverged into an
almost irreconcilable fork of the original code.

Unfortunately, after two decades using this model in the
[RTAI](http://rtai.org) and [Xenomai](https://xenomai.org) projects,
we ended up with a massive issue which is a _severe bit rotting of the
real-time driver stack_. Because the dual kernel ecosystem runs on
very few contributors despite it has many industrial users, every
real-time driver once merged would receive little to no maintenance
effort in the long run. As a result, continuous updates to the
original mainline driver which address bugs, extend support to new
hardware models and versions, is lost for the real-time driver. In
turn, the consequences for such dual kernel systems are bleak:

- narrower hardware support than mainline drivers have.

- more reliability issues than mainline drivers suffer from.

- high cost of updating the real-time driver variant with the bug
  fixes and improvements available from the original mainline
  implementation, which in most cases discourages potential
  contributors from tackling such a task.

Instead of reinventing the wheel with a separate driver stack, the
option EVL follows builds on these observations:

- we can define the notion of operating mode for most common drivers,
  either in-band or out-of-band. Device probing, initialization,
  changes in basic device management are in-band operations by nature,
  which should never overlap with out-of-band I/O transactions aimed
  at transferring data since the latter are supposed to be expedited
  and time-bound. So both aspects of a real-time capable driver should
  be able to coexist, although active at different times, on a
  mutually exclusive basis.

- we would only need a limited subset of the Linux driver stack to
  deliver ultra-low latency performances via out-of-band
  execution. [DMA]({{< relref "core/oob-drivers/dma.md" >}}),
  [SPI]({{< relref "core/oob-drivers/spi.md" >}}), [GPIO]({{< relref
  "core/oob-drivers/gpio.md" >}}), UART, network interface,
  acquisition cards, CAN and other fieldbus devices are among those
  which are typically involved in systems with real-time requirements.

With this in mind, the idea would be to add out-of-band capabilities
to the existing mainline drivers we are interested in for dealing with
real-time I/O, while keeping the original, in-band support
available. These capabilities would be known from the EVL core, and
exclusively accessible via its API under well-defined conditions.
When sharing the implementation with an existing driver is not
possible, either because adding out-of-band capabilities would be
clumsly, or simply because there is no mainline driver for the target
device (e.g. some custom FPGA), it should be a non-issue to implement
a dedicated real-time driver from scratch, using the [EVL kernel
API]({{< relref "core/kernel-api/_index.md" >}}).

Such approach is not at odds with the motivations which prevailed for
using a strictly separated driver stack though:

- there are two ways the GPOS kernel code can adversely affect the
  real-time behavior of the out-of-band code:

  	 - if the out-of-band code spuriously calls into the common,
           in-band kernel API. In such a case, Dovetail can detect
           whether some code running on the out-of-band stage is
           wrongfully calling into the non real-time code, so such
           fact would hardly go unnoticed. Which API calls are legit
           in the context of out-of-band processing, and which are
           not, is well-defined for EVL and such rules are applicable
           to any out-of-band code regardless of its location. In this
           respect, having a separate implementation for the driver
           stack brings no upside.

	 - if the in-band code changes the hardware state in a way
           which may cause high latency peaks (e.g. lengthy cache
           maintenance operations). In this respect, if a real-time
           capable driver is a fork of a mainline kernel driver, any
           pre-existing issue of that kind in the latter
           implementation would be initially imported into the former,
           by definition. Therefore, a careful audit of the code is
           required in any case, whether it is forked off of a
           mainline driver or we are adding out-of-band capabilities
           to the original driver directly.

- there is no reason for the out-of-band services available from a
  common driver to underperform if the implementation clearly gives
  exclusive access to the hardware to the EVL core while out-of-band
  requests are being processed. For instance, reserving exclusive
  access to a SPI bus for out-of-band transfers only until we are done
  with an entire session is an acceptable restriction on in-band usage
  for the purpose of delivering real-time performances. Once such
  out-of-band runtime mode is left, the driver becomes usable anew for
  regular, in-band operations. In some cases, I/O transaction
  management might also be entirely left to the out-of-band code,
  proxying in-band I/O requests via some low-priority queue which
  would be processed by the former on idle time, provided we
  can still meet the real-time requirements doing so.

- although the risk of merge or logic conflicts does exist by
  definition as the extended driver is rebased over later kernel
  releases, it seems an acceptable burden compared to bit rotting of a
  separate code base, which is definitely not. How soundly the
  out-of-band support is integrated into the original driver code will
  make a difference when it comes to rebasing it.

Would such integrated approach cover all the needs for real-time I/O
in a dual kernel system such as EVL? Certainly no. Typically, when
drivers are endpoints of a complex protocol stack such as an IP
network stack attached to network interfaces, the issue of handling
such protocol within a bounded execution time would still not be
solved. In that particular case, a separate real-time capable IP stack
is going to be needed too. However, finding a proper way to extend
existing NIC drivers to serve as endpoints in this real-time IP stack
seems a more tractable problem than maintaining a truckload of
functionally redundant, separate drivers like
[RTnet](https://gitlab.denx.de/Xenomai/xenomai/-/wikis/RTnet)
currently requires.

### Why doesn't EVL implement RTDM?

[RTDM](https://xenomai.org/documentation/xenomai-3/html/xeno3prm/group__rtdm.html)
as an abstract driver model for dual kernel systems was aimed at
addressing two major issues with the latter in the early days:

- provide a common call interface between applications and the
  real-time core, in order to replace the variety of ad hoc mechanisms
  which application developers came up with over time. Some would use
  FIFOs, others shared memory, others some specific system call only
  available from a given flavour of dual kernel system, and so
  on. Every application would come with its own I/O interface to the
  kernel, which was kind of weird. RTDM replaced all of them by a
  single POSIXish API, which strives to mimic the common
  character-based interface, and socket API.

- establish a common kernel API which all RTDM drivers would use, so
  that they would be portable across multiple flavours of dual kernel
  systems implementing the RTDM interface.

Since EVL extends the regular Linux [character-based I/O
interface]({{< relref "core/kernel-api/file/_index.md" >}}) between
applications and real-time drivers, there is no need for any
additional API. Although the question of how to best support the
[socket](http://man7.org/linux/man-pages/man2/socket.2.html) semantics
is still work-in-progress with EVL, implementation-wise there may be
several options. For instance, RTDM implements the socket API as a
character-based interface internally: each socket-related call from
the application branches directly to a dedicated operation handler in
a driver, and data is exchanged between both parties within
per-request buffers; there is no complex logic for managing queues of
socket buffers, no packet reassembly, no design for layered protocols
in between. As a result, the so-called _named_ and _protocol_ driver
interfaces RTDM defines are very close implementation-wise, only
branching to distinct in-kernel handlers, except for the information
identifying the driver to create a channel on
(i.e. [open()](http://man7.org/linux/man-pages/man2/open.2.html) vs
[socket()](http://man7.org/linux/man-pages/man2/socket.2.html)), and
some ancilliary data which can be attached to I/O requests
(e.g. `struct msghdr`) only using the socket API. If this is still the
best option, EVL should do the same, exporting a dedicated socket-like
API to applications. This question is not sorted out yet.

Because RTDM enshrines the notion that a dual kernel system should
provide for its own driver stack aside of the Linux driver model, it
opposes in principle to what EVL aims at.  Achieving a closer
integration of real-time I/O support into the mainline kernel code
whenever possible is a fundamental goal of EVL.

As a consequence, there is not much point in implementing the RTDM
interface over EVL, except maybe as a compatibility layer for porting
[RTAI](http://rtai.org) or [Xenomai](https://xenomai.org)-originated
drivers, although this would be easily done by converting them
directly to the [EVL kernel API]({{< relref
"core/kernel-api/_index.md" >}}) without needing any wrapper of the
kind. As a matter of fact, the EVL kernel API and the portion of the
[Cobalt core
API](https://xenomai.org/documentation/xenomai-3/html/xeno3prm/group__cobalt__core.html)
which has been used as a reference when designing RTDM share all key
semantics.

### Out-of-band capable I/O drivers {#oob-driver-list}

This table gives an overview of the current support for real-time I/O
in EVL, which is not much yet, but poised to improve since the
infrastructure is ready:

<div>
<style>
#oobdrv {
       width: 70%;
       align: left;
}
#oobdrv td:nth-child(1) {
       text-align: center;
}
#oobdrv th:nth-child(1) {
       text-align: center;
}
</style>
<table id="oobdrv">
  <col width="10%">
  <col width="90%">
  <tr>
    <th>Device Type</th>
    <th>Support</th> 
  </tr>
  <tr>
    <td>DMA</td>
    <td>bcm2835-dma, imx-sdma</td> 
  </tr>
  <tr>
    <td>SPI</td>
    <td>spi-bcm2835</td> 
  </tr>
  <tr>
    <td>GPIO <sup>*</sup></td>
    <td>gpio-mxc</td> 
  </tr>
</table>
</div>

<sup>*</sup> In theory, any GPIO pin controller based on the generic
GPIOLIB services should be real-time ready since the latter implements
the out-of-band interface for all of them, at the exception of
controllers which can sleep from their `->get()` and `->set()` pin
state handlers. In practice, each controller we may want to use in
such context should still be audited though, in order to make sure
that no in-band service (e.g. common non-raw spinlocks) is called
under the hood by these handlers. Only the controllers which are known
to work in out-of-band mode _at the time of this writing_ are listed
in the table.

---

{{<lastmodified>}}
