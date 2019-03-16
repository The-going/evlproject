---
title: "Evenless"
---

# Evenless - Dual Kernel Rebooted

## What, Why, and How?

For some classes of application, offloading a particular set of
time-critical tasks to an autonomous core which is hosted by the main
Linux kernel may deliver better performance for a lesser engineering
cost, compared to forcing the entire kernel system to meet the most
stringent requirements only those tasks have. _Evenless_ (EVL for
short) is a practical work on finding the best possible integration of
such a dedicated software core into the mainline kernel. It reboots
the venerable dual kernel design, this time starting from the main
kernel perspective, not from the foreign code extension one would slap
on top of it.

The key issue is about defining a new execution stage in the mainline
kernel logic which would represent high priority, out-of-band
activities, separated from the common work running in-band.  Doing so
requires a few kernel subsystems to know intimately about such a
stage. An architecture-independent interface should enable any flavour
of autonomous cores to run over the out-of-band stage the main kernel
defines. This is the goal of the [Dovetail]({{% relref
"dovetail/_index.md" %}}) interface which is being worked out.

Then, we need a small autonomous core showcasing [Dovetail]({{% relref
"dovetail/_index.md" %}}). Such core should deliver reliable
low-latency services to applications which have to meet real-time
requirements. Last but not least, it should be developed like any
ordinary feature of the mainline kernel, making the best of the rich
infrastructure we have there for improving the integration. This is
the point of the [EVL core]({{%relref "core/_index.md" %}}).

At the end of the day, the success criteria for EVL should be about
exhibiting:

- low engineering and maintenance costs, so that common kernel
  knowledge with limited manpower should be enough to maintain
  [Dovetail]({{% relref "dovetail/_index.md" %}}) and the [EVL
  core]({{%relref "core/_index.md" %}}) over the development tip of
  the mainline kernel.

- low runtime cost, with excellent real-time performances including on
  low-end and mid-range hardware, leaving plenty of cycles for running
  GPOS work concurrently.

- high scalability, from single core to dozens of them happily running
  the time-critical workload in parallel with reliably low latency
  footprint.

## Getting the sources

You need to clone two GIT repositories:

- [Dovetail](({{% relref "dovetail/_index.md" %}})) and the [EVL
core]({{% relref "core/_index.md" %}}) are maintained in separate
branches of the same GIT repository, tracking the [mainline
kernel](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git):

  * git://git.evenless.org/linux-evl.git
  * https://git.evenless.org/linux-evl.git

- the second repository contains the source code of the [EVL
  library]({{% relref "core/user-api/_index.md" %}}), which is the
  user-space API to the EVL core:

  * git://git.evenless.org/libevl.git
  * https://git.evenless.org/libevl.git

The build recipe is available [there]({{% relref
"core/build-steps/_index.md" %}}).

## Licensing terms

SPDX license identifiers are used throughout the code to state the
licensing terms of each file clearly. This boils down to:

- [GPL-2.0](https://spdx.org/licenses/GPL-2.0.html) for the EVL core
  in kernel space.

- [GPL-2.0] (https://spdx.org/licenses/GPL-2.0.html) WITH
  [Linux-syscall-note](https://spdx.org/licenses/Linux-syscall-note.html)
  for the UAPI bits exported to user-space, so that `libevl` knows at
  build time about the ABI details of the system call interface
  implemented by the EVL core.

- [MIT](https://spdx.org/licenses/MIT.html) for all code from
  `libevl`, including the system call wrappers, utilities and test
  programs.
