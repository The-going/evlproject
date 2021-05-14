---
menuTitle: "Ports"
title: "Status of EVL ports"
weight: 20
pre: "&#9656; "
---

To get EVL running on a platform, we need the following software to be
ported in the following sequence:

1. the [Dovetail]({{%relref "dovetail/_index.md" %}}) interface. This
   task is composed of two incremental milestones: first getting the
   [interrupt pipeline]({{% relref "dovetail/pipeline/_index.md" %}})
   to work, then enabling the [alternate scheduling]({{% relref
   "dovetail/altsched/_index.md" %}}). Porting [Dovetail]({{%relref
   "dovetail/_index.md" %}}) is where most of the work takes place,
   the other porting tasks are comparatively quite simple.

2. the [EVL core]({{%relref "core/_index.md" %}}). It is mostly
   composed of architecture-independent code, only a few bits need to
   be ported (a FPU test helper for the most part).

3. the [EVL library]({{%relref "core/user-api/_index.md"
   %}}). Likewise, this code has very little dependencies on the
   underlying CPU architecture and platform. A port boils down to
   resolving the address of the `clock_gettime()` helper in the vDSO
   basically.

The table below summarizes the current status of the existing ports.

> Current target kernel release

Linux 5.1-rc2

> ARM64 SoC

<table class="status" style="width:50%">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>SoC (Board)</th>
    <th>IRQ pipeline <sup>1</sup></th> 
    <th>Alternate task control</th>
    <th>EVL base<sup>2</sup></th>
    <th>EVL stress<sup>3</sup></th>
    <th>Test kernel</th>
  </tr>
  <tr>
    <td><a href="https://developer.qualcomm.com/hardware/dragonboard-410c" target="_blank">DragonBoard 410c</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>5.0</td>
  </tr>
  <tr>
    <td><a href="https://www.raspberrypi.org/products/raspberry-pi-3-model-b/" target="_blank">BCM2837 (Raspberry 3 Model B)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>5.0</td>
  </tr>
  <tr>
    <td><a href="https://www.96boards.org/product/hikey/" target="_blank">Kirin 620 (HiKey LeMaker)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>5.0</td>
  </tr>
  <tr>
    <td><a href="https://wiki.qemu.org/Documentation/Platforms/ARM#Generic_ARM_system_emulation_with_the_virt_machine" target="_blank">virt (QEMU)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>5.1-rc2</td>
  </tr>
</table>

> ARM SoC

<table class="status" style="width:50%">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>SoC (Board)</th>
    <th>IRQ pipeline <sup>1</sup></th> 
    <th>Alternate task control</th>
    <th>EVL base<sup>2</sup></th>
    <th>EVL stress<sup>3</sup></th>
    <th>Test kernel</th>
  </tr>
  <tr>
    <td><a href="https://www.nxp.com/support/developer-resources/hardware-development-tools/sabre-development-system/sabre-board-for-smart-devices-based-on-the-i.mx-6quadplus-applications-processors:RD-IMX6QP-SABRE" target="_blank">i.MX6qp (SabreSD)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>5.0</td>
  </tr>
  <tr>
    <td><a href="https://www.nxp.com/support/developer-resources/hardware-development-tools/sabre-development-system/sabre-board-for-smart-devices-based-on-the-i.mx-7dual-applications-processors:MCIMX7SABRE" target="_blank">i.MX7D (SabreSD)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>5.0</td>
  </tr>
  <tr>
    <td><a href="https://www.96boards.org/documentation/consumer/b2260/hardware-docs/" target="_blank">Cannes2-STiH410 (B2260)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>5.0</td>
  </tr>
  <tr>
    <td><a href="https://www.altera.com/products/soc/portfolio/cyclone-v-soc/overview.html" target="_blank">Cyclone V SoC FPGA (DevKit)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>5.0</td>
  </tr>
  <tr>
    <td><a href="https://www.altera.com/products/soc/portfolio/cyclone-v-soc/overview.html" target="_blank">AM335x-GP (BeagleBone Black)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/unchecked.png"></td>
    <td>5.1-rc2</td>
  </tr>
</table>

<br>

<sup>1</sup> Means that the pipeline torture tests pass (see
`CONFIG_IRQ_PIPELINE_TORTURE_TEST`). This milestone guarantees that we
can deliver high-priority interrupt events immediately to a guest
core, regardless of the work ongoing for the main kernel.

<sup>2</sup> When this box is checked, [EVL]({{%relref
"core/_index.md" %}})'s basic functional test suite runs properly on
the platform, which is a good starting point. So far so good.

<sup>3</sup> When this box is checked, the [EVL core]({{%relref
"core/_index.md" %}}) passes a massive stress test involving the
_hectic_ and _latmus_ applications running in parallel for 24 hrs, all
glitchlessly. This denotes a reliable state, including flawless
alternate scheduling of threads between the main kernel and EVL. On
the contrary, a problem with sharing the FPU unit properly between the
in-band and out-of-band execution contexts is most often the reason
for keeping this box unchecked until the situation is fixed.
