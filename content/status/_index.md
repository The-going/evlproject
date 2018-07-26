---
title: "Status"
date: 2018-07-02T16:47:21+02:00
weight: 4
draft: false
---

This page summarizes the status of the ongoing ports of Dovetail.

> Target kernel release

4.18-rc6

> ARM SoC

<table class="status" style="width:50%">
  <col width="40%">
  <col width="20%">
  <col width="20%">
  <col width="20%">
  <tr>
    <th>Soc (Board)</th>
    <th>IRQ pipeline <sup>1</sup></th> 
    <th>Alternate task control</th>
    <th>Steely<sup>2</sup></th>
  </tr>
  <tr>
    <td align="left"><a href="https://www.nxp.com/support/developer-resources/hardware-development-tools/sabre-development-system/sabre-board-for-smart-devices-based-on-the-i.mx-6quadplus-applications-processors:RD-IMX6QP-SABRE" target="_blank">i.MX6qp (SabreSD)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
  </tr>
  <tr>
    <td align="left"><a href="https://www.nxp.com/support/developer-resources/hardware-development-tools/sabre-development-system/sabre-board-for-smart-devices-based-on-the-i.mx-7dual-applications-processors:MCIMX7SABRE" target="_blank">i.MX7D (SabreSD)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
  </tr>
  <tr>
    <td align="left"><a href="https://www.96boards.org/documentation/consumer/b2260/hardware-docs/" target="_blank">Cannes2-STiH410 (B2260)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
  </tr>
  <tr>
    <td align="left"><a href="https://www.altera.com/products/soc/portfolio/cyclone-v-soc/overview.html" target="_blank">Cyclone V SoC FPGA (DevKit)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
  </tr>
</table>

> ARM64 SoC

<table class="status" style="width:50%">
  <col width="40%">
  <col width="20%">
  <col width="20%">
  <col width="20%">
  <tr>
    <th>Soc (Board)</th>
    <th>IRQ pipeline <sup>1</sup></th> 
    <th>Alternate task control</th>
    <th>Steely<sup>2</sup></th>
  </tr>
  <tr>
    <td align="left"><a href="https://wiki.qemu.org/Documentation/Platforms/ARM#Generic_ARM_system_emulation_with_the_virt_machine" target="_blank">virt (QEMU)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/unchecked.png"></td>
    <td><img src="/images/unchecked.png"></td>
  </tr>
  <tr>
    <td align="left"><a href="https://www.raspberrypi.org/products/raspberry-pi-3-model-b/" target="_blank">BCM2837 (Raspberry 3 Model B)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/unchecked.png"></td>
    <td><img src="/images/unchecked.png"></td>
  </tr>
  <tr>
    <td align="left"><a href="https://www.96boards.org/product/hikey/" target="_blank">Kirin 620 (HiKey LeMaker)</a></td>
    <td><img src="/images/wip.png"></td> 
    <td><img src="/images/unchecked.png"></td>
    <td><img src="/images/unchecked.png"></td>
  </tr>
</table>

<br>

<sup>1</sup> Means that we pass the pipeline torture test (see
`CONFIG_IRQ_PIPELINE_TORTURE_TEST`).

<sup>2</sup> [_Steely_](https://lab.xenomai.org/linux-steely.git/) is
an actively developed [Xenomai
Cobalt](https://xenomai.org/gitlab/xenomai/) derivative we have been
using as a workhorse (or guinea pig at times) for developing and
improving Dovetail.  Steely is nowhere near production software, we
try fundamental changes there for going beyond Cobalt's current
limitations, which could not be merged into the mature Xenomai code
base at this point. When this box is checked, the corresponding
Dovetail port has been tested successfully with a _Steely_ real-time
core.
