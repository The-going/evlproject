---
title: "Evenless"
---

# Evenless

## Dual Kernel Rebooted

For certain types of applications, offloading a particular set of
time-critical tasks to an autonomous core hosted by the Linux kernel
may deliver the best possible performance when compared to forcing the
entire Linux kernel to meet the most stringent scheduling latency
requirements that only those tasks may have. Evenless (EVL for short)
is a practical work on finding the best possible integration of such a
dedicated software core into the mainline Linux kernel.

EVL re-imagines the venerable dual kernel design by introducing the
dual kernel support logic at the heart of the Linux kernel. Because
Linux should define the set of rules to host a guest core, EVL has
been designed specifically with this integration in mind.

The key technical issue is about defining a new execution stage in the
mainline kernel logic which would represent high priority, out-of-band
activities separated from the common work running in-band.  This
requires a few kernel subsystems to know intimately about the new
execution context, which in turn has the following upsides:

- the integration is simpler and cleaner, because we don't need
  sideways. The point is not about hiding the dual kernel interface
  from the main logic, but on the contrary to make it a customary
  interface of the main kernel.

- compared to the
  [I-pipe](https://git.xenomai.org/xenomai/wikis/Getting_The_I_Pipe_Patch)
  which is the interface currently used by several dual kernel systems
  such as [Xenomai](https://xenomai.org/), maintaining the new
  [Dovetail dual kernel interface]({{% relref "dovetail/_index.md"
  %}}) out-of-tree proved to be a much easier task already, without
  fundamental conflicts with upstream changes, only marginal
  adjustments so far.

## Dual kernel made easy

EVL as a project is about enabling engineers to implement their own
flavour of software core running symbiotically with Linux, whatever
the specific purpose of this core may be. This ongoing work is
composed of:

- the [Dovetail]({{% relref "dovetail/_index.md" %}}) interface, which
  introduces a high-priority execution stage for the main kernel,
  enabling an independent software core to run on it.

- the [EVL core]({{%relref "core/_index.md" %}}), a compact autonomous
  core showcasing [Dovetail]({{% relref "dovetail/_index.md" %}}). It
  aims at delivering reliable low-latency services to applications
  which have to meet real-time requirements. It is developed like any
  ordinary feature of the mainline kernel, making the best of the rich
  infrastructure we have there for improving the integration. This
  core is conceived as a learning tool for discovering the dual kernel
  technology as much as it aims at excellent performance and
  usability.

- an in-depth documentation which covers both Dovetail and the EVL
  core, with many cross-references between them, so that people can
  implement their software core of choice almost by example.

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
  `libevl`, which implements the EVL system call wrappers, a few
  utilities and test programs.
