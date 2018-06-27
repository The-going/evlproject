---
title: "Status"
date: 2018-07-02T16:47:21+02:00
weight: 4
draft: false
---

This page summarizes the status of the ongoing ports of Dovetail.

> ARM SoC

| SoC | IRQ pipeline <sup>1</sup> | Alternate task control | Steely<sup>2</sup> |
|:--- |:---:|:---:|:---:|
| [i.MX6qp](https://www.nxp.com/support/developer-resources/hardware-development-tools/sabre-development-system/sabre-board-for-smart-devices-based-on-the-i.mx-6quadplus-applications-processors:RD-IMX6QP-SABRE) | Y | Y | Y |
| [i.MX7d](https://www.nxp.com/support/developer-resources/hardware-development-tools/sabre-development-system/sabre-board-for-smart-devices-based-on-the-i.mx-7dual-applications-processors:MCIMX7SABRE) | Y | Y | Y |

> ARM64 SoC

| SoC | IRQ pipeline | Alternate task control | Steely |
|:--- |:---:|:---:|:---:|
| [QEMU/virt](https://wiki.qemu.org/Documentation/Platforms/ARM#Generic_ARM_system_emulation_with_the_virt_machine) | Y | - | - |
| [HiKey LeMaker](https://www.96boards.org/product/hikey/) | - | - | - |

___

<sup>1</sup> Means that we pass the pipeline torture test (see
`CONFIG_IRQ_PIPELINE_TORTURE_TEST`).

<sup>2</sup> [_Steely_](https://lab.xenomai.org/linux-steely.git/) is
an actively developed [Xenomai
Cobalt](https://xenomai.org/gitlab/xenomai/) derivative we have been
using as a workhorse (or guinea pig at times) for developing and
improving Dovetail.  Steely is nowhere near production software, we
try fundamental changes there for going beyond Cobalt's current
limitations, which could not be merged into the mature Xenomai code
base at this point.
