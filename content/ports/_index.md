---
menuTitle: "Ports"
title: "Status of EVL ports"
weight: 20
pre: "&#8226; "
---

{{% notice info %}}
The EVL project primarily tracks the [mainline Linux
kernel](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git). In
absence of any specific information, all kernel releases mentioned
below refer to mainline Linux.
{{% /notice %}}

To get EVL running on a platform, we need the following software to be
ported in the following sequence:

1. the [Dovetail]({{%relref "dovetail/_index.md" %}}) interface. This
   task is composed of two incremental milestones: first getting the
   [interrupt pipeline]({{% relref "dovetail/pipeline/_index.md" %}})
   to work, then enabling the [alternate scheduling]({{% relref
   "dovetail/altsched.md" %}}). Porting [Dovetail]({{%relref
   "dovetail/_index.md" %}}) is where most of the work takes place,
   the other porting tasks are comparatively quite simple.

2. the [EVL core]({{%relref "core/_index.md" %}}) which is mostly
   composed of architecture-independent code, therefore only a few
   bits need to be ported (a FPU test helper for the most part).

3. the [EVL library]({{%relref "core/user-api/_index.md"
%}}). Likewise, this code has very little dependencies on the
underlying CPU architecture and platform. A port boils down to
resolving the address of the
[clock_gettime(3)](http://man7.org/linux/man-pages/man3/clock_gettime.3.html)
in the vDSO.

The table below summarizes the current status of the EVL ports to
particular SoCs which we are aware of. If you are interested in
porting your own autonomous core to a particular kernel release
Dovetail supports, you certainly need the _IRQ pipeline_ column
matching the target platform to be checked, and likely the _Alternate
scheduling_ column as well. Running the EVL core requires both
features to be available. If you ported EVL to a SoC which does not
appear in this list and want to let people know about it,
please drop me a note at <rpm@xenomai.org>.

> Current target kernel release

Linux {{< param evlTrackingKernel >}}

> ARM64 SoC

<div>
<style>
#ports {
       width: 70%;
}
#ports td {
       text-align: center;
}
</style>

<table id="ports">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>SoC (Board)</th>
    <th>IRQ pipeline<sup>1</sup></th> 
    <th>Alternate scheduling</th>
    <th>EVL base<sup>2</sup></th>
    <th>EVL stress<sup>3</sup></th>
    <th>Test kernel</th>
  </tr>
  <tr>
    <td><a href="https://www.qualcomm.com/products/qcs404/" target="_blank">Qualcomm QCS404</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.1-rc3</td>
  </tr>
  <tr>
    <td><a href="https://developer.qualcomm.com/hardware/dragonboard-410c" target="_blank">Qualcomm DragonBoard 410c</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.0</td>
  </tr>
  <tr>
    <td><a href="https://www.raspberrypi.org/products/raspberry-pi-3-model-b/" target="_blank">Broadcom BCM2837 (Raspberry 3 Model B)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc2</td>
  </tr>
  <tr>
    <td><a href="https://www.raspberrypi.org/products/raspberry-pi-4-model-b/" target="_blank">Broadcom BCM2711 (Raspberry PI 4 Model B)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc2</td>
  </tr>
  <tr>
    <td><a href="https://www.96boards.org/product/hikey/" target="_blank">HiSilicon Kirin 620 (HiKey LeMaker)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.5-rc7</td>
  </tr>
  <tr>
    <td><a href="https://www.xilinx.com/support/documentation/boards_and_kits/zcu102/ug1182-zcu102-eval-bd.pdf" target="_blank">Xilinx Zynq UltraScale+ (ZCU102)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.2</td>
  </tr>
  <tr>
    <td><a href="https://www.variscite.com/product/system-on-module-som/cortex-a53-krait/dart-mx8m-mini-nxp-i-mx8m-mini/" target="_blank">NXP i.MX8M Mini (Variscite DART-MX8M-MINI)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.2.9 <sup>*</sup></td>
  </tr>
  <tr>
    <td><a href="https://wiki.qemu.org/Documentation/Platforms/ARM#Generic_ARM_system_emulation_with_the_virt_machine" target="_blank">QEMU virt</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc2</td>
  </tr>
</table>

<sup>*</sup> Mainline kernel with SoC-specific bits picked from the vendor tree.

> ARM SoC

<table id="ports">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>SoC (Board)</th>
    <th>IRQ pipeline<sup>1</sup></th> 
    <th>Alternate scheduling</th>
    <th>EVL base<sup>2</sup></th>
    <th>EVL stress<sup>3</sup></th>
    <th>Test kernel</th>
  </tr>
  <tr>
    <td><a href="https://www.nxp.com/support/developer-resources/hardware-development-tools/sabre-development-system/sabre-board-for-smart-devices-based-on-the-i.mx-7dual-applications-processors:MCIMX7SABRE" target="_blank">NXP i.MX7D (SabreSD)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.2</td>
  </tr>
  <tr>
    <td><a href="https://www.nxp.com/support/developer-resources/hardware-development-tools/sabre-development-system/sabre-board-for-smart-devices-based-on-the-i.mx-6quadplus-applications-processors:RD-IMX6QP-SABRE" target="_blank">NXP i.MX6qp (SabreSD)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc2</td>
  </tr>
  <tr>
    <td><a href="https://beagleboard.org/black/" target="_blank">TI AM335x-GP (BeagleBone Black)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.1-rc3</td>
  </tr>
  <tr>
    <td><a href="https://www.altera.com/products/soc/portfolio/cyclone-v-soc/overview.html" target="_blank">Altera Cyclone V SoC FPGA (DevKit)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.6</td>
  </tr>
  <tr>
    <td><a href="https://www.96boards.org/documentation/consumer/b2260/hardware-docs/" target="_blank">STMicro Cannes2-STiH410 (B2260)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.2-rc7</td>
  </tr>
  <tr>
    <td><a href="https://www.raspberrypi.org/products/raspberry-pi-2-model-b/" target="_blank">Broadcom BCM2636 (Raspberry PI 2 Model B)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc2</td>
  </tr>
  <tr>
    <td><a href="http://nanopi.io/nanopi-neo.html/" target="_blank">AllWinner H3 (NanoPI NEO)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.5-rc2</td>
  </tr>
</table>

> X86_64

<table id="ports">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Chipset (Module)</th>
    <th>IRQ pipeline<sup>1</sup></th> 
    <th>Alternate scheduling</th>
    <th>EVL base<sup>2</sup></th>
    <th>EVL stress<sup>3</sup></th>
    <th>Test kernel</th>
  </tr>
  <tr>
    <td><a href="https://www.linux-kvm.org/page/Main_Page" target="_blank">QEMU KVM</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc2</td>
  </tr>
  <tr>
    <td><a href="https://www.tq-group.com/en/products/tq-embedded/x86-architecture/tqmxe39m/" target="_blank">Intel Atom x5-E3940 (TQMxE39M)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc2</td>
  </tr>
  <tr>
    <td><a href="https://www.dfi.com/product/index/224#specification" target="_blank">Intel C236 core i7 quad (DFI SD631)</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc1</td>
  </tr>
  <tr>
    <td><a href="https://ark.intel.com/content/www/us/en/ark/products/34687/intel-desktop-board-dq45cb.html" target="_blank">Intel Desktop Board DQ45CB</a></td>
    <td><img src="/images/checked.png"></td> 
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td><img src="/images/checked.png"></td>
    <td>v5.7-rc1</td>
  </tr>
</table>
</div>

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
`hectic` and `latmus` applications running in parallel along with the
full test suite for 24 hrs, all glitchlessly. This denotes a reliable
state, including flawless alternate scheduling of threads between the
main kernel and EVL. On the contrary, a problem with sharing the FPU
unit properly between the in-band and out-of-band execution contexts
is most often the reason for keeping this box unchecked until the
situation is fixed.

---

{{<lastmodified>}}
